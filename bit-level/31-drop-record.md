# 31 — Drop record (bit-level)

- **Source:** [§3.3 Drop records](../wire-format/21-drop-store.md)
- **Total width:** 48 bytes (fixed)
- **Alignment:** immediately follows the slab header (offset 56 within
  the slab); subsequent drop records are packed contiguously in
  declaration order.

Each drop in a slab has a `DropRecord` describing where its bytes
live inside one of the slab's solid windows. Readers use the
`(solid_window_index, offset_in_window, len_in_window)` triple to
extract the drop's bytes from the decompressed solid window.

## Byte-offset table

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 32 | drop_id | bytes | — | `DropId = BLAKE3(plaintext)` (§1.1) |
| 32 | 4 | plaintext_len | u32 | LE | bytes after codec/AEAD inverse; sizes the destination buffer |
| 36 | 3 | representation | bytes | — | `(codec: u8, aead: u8, ec: u8)` per §2.2 |
| 39 | 1 | solid_window_index | u8 | — | index of the solid window containing this drop's bytes |
| 40 | 4 | offset_in_window | u32 | LE | byte offset within the decompressed solid window |
| 44 | 4 | len_in_window | u32 | LE | byte length within the decompressed solid window |

Total: 32 + 4 + 3 + 1 + 4 + 4 = **48 bytes**.

## Bit-position diagram

```
Byte  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     |<------------------------- drop_id (32 bytes) ----------------------->|
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

...  30  31  32  33  34  35  36  37  38  39  40  41  42  43  44  45  46  47
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     |  drop_id (cont'd) |<- pt_len ->|<- repr (3) ->|swi|<- offset ->|<- len --->|
     |                   | u32 LE     | codec aead ec|u8 | u32 LE     | u32 LE    |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
                                                39   40          44          48
```

Each cell is one byte. `swi` is `solid_window_index`.

## Field semantics

### drop_id (bytes 0..32, 32 bytes)

The drop's identity: `BLAKE3(plaintext)` per [§1.1](../concepts/11-identity.md).
Identity is independent of representation choice (§1.3): the same
plaintext sealed with different AEAD or codec settings shares the
same `DropId`.

Readers MUST verify `BLAKE3(decoded_plaintext) == drop_id` after
all representation inversions; the hash check is the canonical test
of identity (§1.1).

### plaintext_len (bytes 32..36, u32 LE)

Byte length of the drop's plaintext AFTER full decoding through
the representation pipeline (codec inverse, AEAD open, EC
reconstruct). Readers use this to size the destination buffer; no
in-band length framing is required.

Maximum value: `u32::MAX` ≈ 4 GiB. The spec constrains drop sizes
to the FastCDC chunk size ceiling (typically 4 MiB for content-defined
chunking or 64 MiB for fixed-size chunking); readers SHOULD reject
values exceeding the configured ceiling as `Corrupt`.

### representation (bytes 36..39, 3 bytes)

The codec, AEAD, and EC ids applied to this drop's bytes. Encoded
per [§2.2](../01-glossary.md):

| Byte | Field | Notes |
|---|---|---|
| 36 | codec | 0x00 = store; 0x01 = lz4 (mandatory baseline); others per [§11 Codec registry](../registries/11-codec.md) |
| 37 | aead | 0x00 = plaintext; 0x01 = XChaCha20-Poly1305 (mandatory baseline); others per [§10 AEAD registry](../registries/10-aead.md) |
| 38 | ec | 0x00 = no EC; 0x01+ per [§16 Erasure coding](../crypto/16-ec.md) |

When the slab's `crypto_hint = 0` (plaintext slab), this record's
`aead` MUST be `0`. When the slab's `ec_descriptor = 0` (no EC),
this record's `ec` MUST be `0`. Mismatches are `Corrupt`.

### solid_window_index (byte 39, u8)

Index of the solid window within the slab containing this drop's
bytes. A slab MAY have multiple solid windows (one per class group
per [§20.1 decision](../decisions/20-resolved.md)). The index is
0-based; values exceeding the slab's actual window count are
`Corrupt`.

### offset_in_window (bytes 40..44, u32 LE)

Byte offset of this drop's bytes within the decompressed solid
window identified by `solid_window_index`. The offset is relative
to the start of the window's plaintext content, not to the slab's
header.

### len_in_window (bytes 44..48, u32 LE)

Byte length of this drop's representation within the decompressed
solid window. For codec = store (0x00), `len_in_window` equals
`plaintext_len`. For other codecs, `len_in_window` is the
compressed length; `plaintext_len` is the post-decompression
length.

Readers MUST verify `offset_in_window + len_in_window` does not
exceed the solid window's total length. Out-of-bounds reads are
`Corrupt`.

## Validation rules

A `DropRecord` is structurally valid iff:

1. The input is at least 48 bytes long. Shorter inputs return
   `CoreError::TooShort { have, need: 48 }`.
2. `plaintext_len` does not exceed the configured drop-size ceiling
   (default 64 MiB). Otherwise `Corrupt`.
3. `representation.aead` is `0x00` if the slab's `crypto_hint` is
   `0x00` (plaintext slab). Mismatch: `Corrupt`.
4. `representation.ec` is `0x00` if the slab's `ec_descriptor` is
   `0x00` (no EC slab). Mismatch: `Corrupt`.
5. `offset_in_window + len_in_window` does not overflow u32. If it
   does: `Corrupt`.
6. `solid_window_index` is within the slab's solid-window count
   (known after parsing all drop records). If not: `Corrupt`.

Cross-window bounds checks (rule 6, and the offset+len check
against the window's actual length) require knowledge of the
slab's full structure; they run after the drop-record array is
parsed. The parser of an individual `DropRecord` performs only
the self-contained checks (rules 1–5).

## Worked example

### Store-coded plaintext drop

A 1 024-byte plaintext drop stored at offset 0 of solid window 0:

```
00 11 22 33  45 56 67 78  ... (32 bytes total) ...  ff  (drop_id)
00 04 00 00                                              plaintext_len = 1024
00 00 00                                                 codec=store, aead=none, ec=none
00                                                       solid_window_index = 0
00 00 00 00                                              offset_in_window = 0
00 04 00 00                                              len_in_window = 1024
```

48 bytes total. Since codec = store, `len_in_window` equals
`plaintext_len`.

### LZ4-compressed drop in a sealed slab

A 4 096-byte plaintext drop LZ4-compressed to 1 024 bytes, sealed
with XChaCha20-Poly1305, in solid window 1 at offset 2 048:

```
12 34 56 ... 32 bytes total ... cd  ef                  (drop_id)
00 10 00 00                                              plaintext_len = 4096
01 01 00                                                 codec=lz4, aead=XChaCha20-Poly1305, ec=none
01                                                       solid_window_index = 1
00 08 00 00                                              offset_in_window = 2048
00 04 00 00                                              len_in_window = 1024 (compressed)
```

Note `len_in_window` (1 024) ≠ `plaintext_len` (4 096) because LZ4
compressed the bytes; the reader decompresses 1 024 bytes to obtain
4 096 bytes of plaintext, then verifies `BLAKE3(plaintext) == drop_id`.

## Cross-references

- [§3.3 Drop records](../wire-format/21-drop-store.md) — semantic
  description.
- [§3.4 Solid windows](../wire-format/21-drop-store.md) — what the
  `solid_window_index` references.
- [§1.1 The identity rule](../concepts/11-identity.md) — `DropId`
  and the post-decode verification.
- [§2.2 Semantic types](../01-glossary.md) — `DropId`,
  `Representation` definitions.
- [§10 AEAD registry](../registries/10-aead.md), [§11 Codec
  registry](../registries/11-codec.md), [§16 Erasure coding](../crypto/16-ec.md)
  — id spaces referenced by `representation`.
- [30-slab-header.md](30-slab-header.md) — the slab header that
  precedes drop records in every slab.
