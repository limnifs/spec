# 32 — Representation triple (bit-level)

- **Source:** [§2.2 Semantic types](../01-glossary.md)
- **Total width:** 3 bytes (fixed)
- **Alignment:** none (the triple is embedded inside larger structures:
  drop record §3.3, future representation-plane records).

A `Representation` triple identifies how a particular byte sequence
was encoded. It never affects identity (§1.3): two representations
of the same plaintext share the same `DropId`. The triple records
the codec, AEAD, and EC scheme a writer applied, so a reader knows
which inverse transforms to invoke during decoding.

## Byte-offset table

| Offset | Width | Field | Type | Notes |
|---|---|---|---|---|
| 0 | 1 | codec | u8 | 0x00 = store; 0x01 = lz4 (mandatory); others per [§11 Codec registry](../registries/11-codec.md) |
| 1 | 1 | aead | u8 | 0x00 = plaintext; 0x01 = XChaCha20-Poly1305 (mandatory); others per [§10 AEAD registry](../registries/10-aead.md) |
| 2 | 1 | ec | u8 | 0x00 = no EC; 0x01+ per [§16 Erasure coding](../crypto/16-ec.md) |

Total: **3 bytes**.

The order is fixed: codec first, then AEAD, then EC. Readers MUST
consume fields in this order; writers MUST emit in this order.

## Bit-position diagram

```
Byte  0    1    2
     +----+----+----+
     | cc | aa | ee |
     +----+----+----+
     |<codec>|<aead>|<ec>|
```

Each cell is one byte.

## Field semantics

### codec (byte 0, u8)

The codec id selecting how the bytes were compressed. The codec is
the first inverse transform applied during decoding: readers
decompress the bytes via the codec before any AEAD open or EC
reconstruction.

| Value | Meaning |
|---|---|
| `0x00` | Store (no compression; bytes are the literal plaintext) |
| `0x01` | LZ4 (mandatory baseline) |
| `0x02`–`0xFE` | Other codecs per [§11 Codec registry](../registries/11-codec.md) (zstd, brotli, lzma, etc.) |
| `0xFF` | Extended (post-v1; readers reject with `UnsupportedFeature`) |

For codec = store (`0x00`), the reader skips decompression entirely.

### aead (byte 1, u8)

The AEAD id selecting how the bytes were sealed. The AEAD open is
the second inverse transform during decoding (after codec
inverse, before EC reconstruct).

| Value | Meaning |
|---|---|
| `0x00` | Plaintext (no AEAD; bytes are unsealed) |
| `0x01` | XChaCha20-Poly1305 (mandatory baseline) |
| `0x02`–`0xFE` | Other AEAD per [§10 AEAD registry](../registries/10-aead.md) |
| `0xFF` | Extended (post-v1; readers reject with `UnsupportedFeature`) |

For aead = plaintext (`0x00`), the reader skips the open step. The
actual key (when `aead != 0`) lives in the manifest's crypto params
section (§5.5), NOT in this triple.

### ec (byte 2, u8)

The erasure-coding id selecting how the bytes were sharded. The EC
reconstruct is the third inverse transform during decoding.

| Value | Meaning |
|---|---|
| `0x00` | No EC (no sharding) |
| `0x01`–`0xFE` | Reed-Solomon scheme per [§16 Erasure coding](../crypto/16-ec.md) |
| `0xFF` | Extended (post-v1; readers reject with `UnsupportedFeature`) |

For ec = none (`0x00`), the reader skips reconstruction.

## Validation rules

A `Representation` triple is structurally valid iff:

1. `codec` is not `0xFF`. Otherwise: `UnsupportedFeature`.
2. `aead` is not `0xFF`. Otherwise: `UnsupportedFeature`.
3. `ec` is not `0xFF`. Otherwise: `UnsupportedFeature`.

Cross-record consistency (e.g., "the slab's `crypto_hint` matches
the representation's `aead`") is a higher-layer concern handled by
the slab-walker. The triple's own parser only checks the
extended-sentinel rules.

## Worked example

### Plaintext store-coded drop (no AEAD, no EC)

The minimal representation used by the smallest v0.1 image:

```
00    00    00
codec aead  ec
store plain none
```

3 bytes. The reader skips every inverse transform and uses the
bytes as-is.

### LZ4-compressed sealed drop with EC

A representation using all three transforms:

```
01    01    01
codec aead  ec
lz4   XCha  RS
```

3 bytes. The reader reconstructs EC shards (if needed), opens the
AEAD, then decompresses LZ4 to obtain the plaintext.

### Extended codec (rejected in v1)

```
FF    00    00
codec aead  ec
ext.  plain none
```

3 bytes. The reader rejects with `UnsupportedFeature("codec 0xFF
(extended, post-v1)")`.

## Cross-references

- [§2.2 Semantic types](../01-glossary.md) — `Representation`
  definition.
- [§1.3 Representation plane separation](../concepts/11-identity.md)
  — why representation never affects identity.
- [§10 AEAD registry](../registries/10-aead.md) — `aead` id space.
- [§11 Codec registry](../registries/11-codec.md) — `codec` id space.
- [§16 Erasure coding](../crypto/16-ec.md) — `ec` id space.
- [31-drop-record.md](31-drop-record.md) — where the triple appears
  inside a drop record (bytes 36..39).
