### 2. Terminology

This section is normative for term usage; the semantic types listed at
the end are normative for byte layout.

#### 2.1 Limnologic vocabulary

The following terms are used throughout this spec with the meanings
given here. The vocabulary is load-bearing, not decorative — conflating
"slice" and "drop" produces an implementation bug class.

- **slice** — A unit of user content presented to the writer pipeline.
  For the common case of a regular file, a slice is a file's content
  (one slice per file). For sparse files, a slice is a contiguous run
  of non-hole bytes. For very small files (size ≤ a spec-pinned
  threshold chosen by `01-flatbuffers-schema`), a slice MAY be stored
  inline in the inode's metadata rather than broken into drops. A slice
  has a fixed byte order; reading a slice in reversed byte order is
  NOT the same slice.
- **drop** — A content-addressed chunk; the atom of storage and the unit
  of identity. A drop's identity is `DropId = BLAKE3(plaintext)` (§1.1).
  A drop is the smallest unit at which deduplication, codec
  compression, AEAD sealing, and EC sharding operate. A drop's
  plaintext length is recorded in metadata; readers use that length to
  size the decode destination buffer (no in-band length framing).
- **slab** — A contiguous byte object in the drop store (Layer 1). A
  slab holds many drops, typically 4 MiB minimum and 64 MiB maximum
  per spec-pinned defaults. A slab has a `SlabId` (§2.2) and a fixed
  byte order. A slab MAY be EC-sharded; the resulting shards are
  themselves not "slabs" in the strict sense (shards are addressed via
  shard records in the slab index, §3).
- **epilimnion** — The hot tier: drops freshly written, encoded with a
  fast codec (LZ4 or store), packed in unconsolidated slabs.
- **metalimnion** — The transition tier: drops being re-encoded by a
  background compactor (the "thermocline" is the moving boundary
  between epilimnion and metalimnion/hypolimnion).
- **hypolimnion** — The cold tier: drops re-encoded with a deep codec
  (zstd default; lzma or brotli for text-heavy classes; store for
  incompressible), packed in consolidated slabs.
- **turnover** — A derivation operation that mixes tiers: flatten an
  overlay chain, defragment slabs, re-encode drops by class, garbage-
  collect unreferenced drops. Produces a standalone image with no
  external slab references (§8.3).
- **meromictic** — An overlay chain that is never flattened (or turned
  over). A meromictic chain is valid long-term state; readers MUST NOT
  require flattening. The word is from limnology: a meromictic lake
  does not fully mix its layers.

#### 2.2 Semantic types

The following types are defined by this spec. Implementations MUST emit
them as distinct types (Rust newtypes, Python dataclasses with frozen
slots, or equivalents) — NOT as bare `bytes`, `int`, or `str` aliases.
Bypassing the type system defeats the per-field semantic constraints
(§1.1 multihash format, §1.4 determinism, etc.).

| Type | Encoding | Width | Notes |
|---|---|---|---|
| `DropId` | 32 bytes raw; `b3:<base32>` text | 256 bits | multihash form for display |
| `SlabId` | 8 bytes per-image ordinal (LE u64) + 32 bytes content hash | 320 bits | ordinal ensures distinct slabs with same hash in same image |
| `ManifestRoot` | 32 bytes raw; same display as `DropId` | 256 bits | BLAKE3 of the Merkle hash list (§5.10) |
| `Tier` | 1 byte enum: `0x00` = epilimnion, `0x01` = metalimnion, `0x02` = hypolimnion | 8 bits | per-slab tier tagging (§3) |
| `Representation` | 3 bytes: codec id (1), aead id (1, `0x00` = none), ec id (1, `0x00` = none) | 24 bits | registry rows in §11, §10, §14 |
| `SlabRef` | `SlabId` (40 bytes) + offset (8 bytes LE u64) + len (8 bytes LE u64) + `Representation` (3 bytes) + locator count (varint) + locator entries | variable | locator entries are the §12 wire form |

`SlabRef` field order is fixed: `SlabId`, `offset`, `len`,
`Representation`, then locator count and locator entries. Readers MUST
consume fields in this order; writers MUST emit in this order. Any
deviation breaks the `DropSource` post-condition (§6.2).

All widths are exact. Implementations MUST NOT widen (e.g., store
`DropId` as 33 bytes with a null terminator); every stored value is
exactly the width listed. Width changes are spec amendments, not
implementation choices.

---
