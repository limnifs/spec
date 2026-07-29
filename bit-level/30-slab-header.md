# 30 — Slab header (bit-level)

- **Source:** [§3.2 SlabHeader](../wire-format/21-drop-store.md)
- **Total width:** 56 bytes (fixed)
- **Alignment:** 4-byte aligned (magic + version field).

The slab header is the fixed-size prefix at offset 0 of every slab
in the drop store. Readers validate the header before parsing any
drop records, solid windows, or EC shards. A header that fails
validation terminates parsing with a structured error — no further
bytes in the slab are read.

## Byte-offset table

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 4 | magic | bytes | — | `LIM1` = `0x4C 0x49 0x4D 0x31` |
| 4 | 2 | format_version | u16 | LE | bump-incompatible layout changes (§17) |
| 6 | 8 | slab_id.ordinal | u64 | LE | per-image ordinal, distinguishes same-hash slabs |
| 14 | 32 | slab_id.hash | bytes | — | BLAKE3 of slab contents (§2.2) |
| 46 | 8 | total_length | u64 | LE | bytes including header |
| 54 | 1 | ec_descriptor | u8 | — | 0 = no EC; 1..N = EC id (§16); 255 = extended (post-v1) |
| 55 | 1 | crypto_hint | u8 | — | 0 = plaintext; N = AEAD id (§10); key lives in manifest §5.5 |

Total: 4 + 2 + 8 + 32 + 8 + 1 + 1 = **56 bytes**.

The `slab_id` is a single semantic type (`SlabId`, §2.2) but its
two sub-fields are encoded sequentially on the wire: ordinal first,
then hash. This matches the field order of the type definition.

## Bit-position diagram

```
Byte  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14 ...
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     | 4C| 49| 4D| 31| vv | vv |<------- slab_id.ordinal (u64 LE) ------>|<- slab_id.hash[0..] ->|
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     |<- magic 'LIM1 ->|<- format_version ->|  per-image ordinal     |  content hash (32 bytes)...
```

```
... 13  14  15  16  17  18  19  20  21  22  23  24  25  26  27  28 ...
    +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    |<---------------- slab_id.hash (32 bytes, cont'd) ---------------->|
    +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
```

```
... 45  46  47  48  49  50  51  52  53  54  55
    +---+---+---+---+---+---+---+---+---+---+---+
    | hash[31] |<----- total_length (u64 LE) ----->| ec| cr|
    +---+---+---+---+---+---+---+---+---+---+---+
                                  8 bytes          1   1
```

Each cell is one byte, shown in hexadecimal. `vv` denotes a version
byte (low byte of the u16 LE).

## Field semantics

### magic (bytes 0..4)

The byte sequence `0x4C 0x49 0x4D 0x31` (ASCII `LIM1`). Lets readers
reject misidentified bytes before any further parse step. The bytes
spell `LIM1` for grepability and to match the file-extension mnemonic
(`.lim` → Limni → `LIM1`, where `1` is the format generation).

Readers MUST reject any other magic with `CoreError::BadMagic { found }`
where `found` is the actual 4-byte sequence encountered.

### format_version (bytes 4..6, u16 LE)

The slab layout version. Currently `1`. Bumped on bump-incompatible
changes to the slab header, drop record (§3.3), solid window layout
(§3.4), or EC shard layout (§3.5).

`format_version` is the per-blob version byte (per [pivot
D3](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md)):
it bumps when this slab's byte layout changes. It is independent of
the manifest's section versions and of the metadata layer's version.

### slab_id.ordinal (bytes 6..14, u64 LE)

Per-image ordinal, distinguishing slabs that hash to the same value
within one image (§2.2). Ordinals are assigned by the writer in
slice-traversal order (lexicographic by slice key per §1.4); they
are not guaranteed to be contiguous (turnover compaction can leave
gaps).

### slab_id.hash (bytes 14..46, 32 bytes)

BLAKE3 hash of the slab's content (drop records + solid windows +
EC shards), excluding the slab header itself. The hash is the
content-addressed part of the SlabId; the ordinal disambiguates
collisions within one image.

### total_length (bytes 46..54, u64 LE)

Total byte length of the slab, including the 56-byte header. Used
by readers to size the slab buffer and by locators (§12) to issue
range requests. Must be ≥ 56 (a slab with no drop records is
structurally valid but degenerate).

The spec constrains slab sizes to the 4 MiB–64 MiB range by default
(§3.1); the manifest's parameters section MAY override (§5.4).
Readers SHOULD reject `total_length` outside the configured range as
`Corrupt`.

### ec_descriptor (byte 54, u8)

Erasure-coding scheme applied to this slab.

| Value | Meaning |
|---|---|
| `0x00` | No erasure coding. Slab carries no parity shards. Readers MUST NOT attempt reconstruction. |
| `0x01`–`0xFE` | EC scheme id, referencing [§16 Erasure coding](../crypto/16-ec.md) and the EC scheme registry. The slab carries `k+m` Reed-Solomon shards per §3.5. |
| `0xFF` | Extended descriptor follows (post-v1; readers in v1 reject with `UnsupportedFeature`). |

### crypto_hint (byte 55, u8)

AEAD scheme applied to this slab's payload (everything after the
header).

| Value | Meaning |
|---|---|
| `0x00` | Plaintext. No AEAD. The manifest's crypto params (§5.5) MUST agree: `image_key_id` is `0` for an all-plaintext image. |
| `0x01`–`0xFE` | AEAD id, referencing [§10 AEAD registry](../registries/10-aead.md). The actual key lives in the manifest (§5.5), NOT in this header — the hint only tells readers which AEAD to invoke. |
| `0xFF` | Extended hint follows (post-v1; readers in v1 reject with `UnsupportedFeature`). |

Readers MUST reject `ec_descriptor` and `crypto_hint` combinations
that the manifest does not declare. Concretely, if the slab says
`crypto_hint = 1` but the manifest's `image_key_id = 0`, the slab
cannot be decrypted; readers reject with `Corrupt`.

## Validation rules

A slab header is valid iff:

1. The input is at least 56 bytes long. Shorter inputs return
   `CoreError::TooShort { have, need: 56 }`.
2. `magic == [0x4C, 0x49, 0x4D, 0x31]`. Otherwise
   `CoreError::BadMagic { found }`.
3. `format_version == 1`. Other versions: `UnsupportedFeature`.
4. `total_length >= 56`. Otherwise `Corrupt`.
5. `total_length` does not exceed the configured slab size ceiling
   (default 64 MiB, §3.1). Otherwise `Corrupt`.
6. `ec_descriptor ∈ {0x00} ∪ [0x01, 0xFE]`. Value `0xFF`:
   `UnsupportedFeature` in v1.
7. `crypto_hint ∈ {0x00} ∪ [0x01, 0xFE]`. Value `0xFF`:
   `UnsupportedFeature` in v1.

Cross-field consistency (e.g., "if `crypto_hint != 0` then
manifest's `image_key_id != 0`") is a higher-layer concern handled
once the manifest's crypto params are parsed. The slab header parser
only checks structural invariants within the slab itself.

## Worked example

### Plaintext slab, no EC, ordinal 1

```
4C 49 4D 31                  01 00                01 00 00 00 00 00 00 00
|<- magic 'LIM1 ->|          | format_version=1 |  |<- slab_id.ordinal = 1 (LE) ->|

DE AD BE EF ... 32 bytes total                     <- slab_id.hash
...
00 00 00 00 00 00 00 00         00 01 00 00 00 00 00 00         00  00
|<- total_length = 256 ->|      (just an example; 256-byte slab)  |EC|CR|
                                                                 none plaintext
```

### EC-enabled slab, k=4 m=2, sealed with AEAD id 1

```
4C 49 4D 31        01 00      07 00 00 00 00 00 00 00       ...hash...
|<- magic 'LIM1 ->| |ver=1 |  |<- slab_id.ordinal = 7 ---->|  (32 bytes)

...hash (cont'd)...

00 40 00 00 00 00 00 00         01  01
|<- total_length = 16384 ->|     |EC|CR|
                                  id=1 AEAD id=1
                                  (RS) (XChaCha20)
```

(16 384 bytes is the minimum EC-enabled slab size; the RS shards
are computed over a 16 384-byte payload split into k=4 data shards
of 4 096 bytes each, plus m=2 parity shards of 4 096 bytes each.)

## Cross-references

- [§3.2 SlabHeader](../wire-format/21-drop-store.md) — semantic
  description.
- [§3.1 Slab format](../wire-format/21-drop-store.md) — slab layout
  overview.
- [§3.3 Drop records](../wire-format/21-drop-store.md) — what comes
  after the header.
- [§2.2 Semantic types](../01-glossary.md) — `SlabId` definition.
- [§10 AEAD registry](../registries/10-aead.md) — `crypto_hint` id
  space.
- [§16 Erasure coding](../crypto/16-ec.md) — `ec_descriptor` id space.
- [§17 Versioning policy](../versioning/17-versioning.md) — what
  bumping `format_version` means.
- [35-manifest-header.md](35-manifest-header.md) — the analogous
  fixed-width header for manifests (same documentation pattern).
