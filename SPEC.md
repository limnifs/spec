# LimniFS Format Specification — v0.1

- **Status:** DRAFT OUTLINE — section structure and key decisions, for review.
  Not yet a complete spec. Each section's prose is filled in by subsequent
  PRs as called out in [01-format-spec-v01](https://github.com/limnifs/limnifs/blob/main/TODO.impl/01-spec/01-format-spec-v01.md).
- **Spec version:** 0.1 (semantic: first verifiable reader target —
  Phase 0 exit gate is two independent readers passing the v0.1 vectors).
- **Supersedes:** none (initial version).
- **Scope:** the wire format of `.limni` images and the behavioral contracts
  required to *read* them. Implementation behavior (chunking, classification,
  deepening policy) is owned by other components and referenced from here.
- **Authority:** where this document and any implementation diverge, this
  document is correct. Where this document and `00-architecture` diverge,
  `00-architecture` wins (architecture specifies interfaces; this specifies
  bytes). Where this document and the design doc diverge, this document
  wins.

## How to read this spec

Normative language follows RFC 2119 (`MUST`, `SHOULD`, `MAY`). Priority
order for resolving questions about the format:

1. This `SPEC.md`.
2. `schema/*.fbs` (added by [01-flatbuffers-schema]).
3. Registry data files under `registries/*` (added by [01-feature-flag-registry]).

Sections marked **(informative)** do not constrain implementations; all
others do.

[01-flatbuffers-schema]: https://github.com/limnifs/limnifs/blob/main/TODO.impl/01-spec/01-flatbuffers-schema.md
[01-feature-flag-registry]: https://github.com/limnifs/limnifs/blob/main/TODO.impl/01-spec/01-feature-flag-registry.md

---

## Part I — Identity and invariants

### 1. Foundational invariants

This section is normative. A reader MUST implement every invariant stated
here. Implementation divergences invalidate spec conformance.

#### 1.1 The identity rule

`DropId` is the BLAKE3-derived identity of a drop's plaintext bytes:

```
DropId       = BLAKE3(plaintext)                      // 256-bit output
DropId.text  = "b3:" || base32_no_pad(DropId)        // multihash display
```

The `plaintext` argument is the drop's bytes AFTER full decoding through
every representation transform (codec inverse, AEAD open, EC
reconstruct). It MUST be the bytes the user would read from the
corresponding slice range if the image were mounted read-only. See
`02-algorithms.md §2` for the full decoding-then-verification sequence.

`DropId` is a 32-byte value (256 bits). Its display form is the multihash
text string `b3:<base32-lower-no-pad>` where `<base32-lower-no-pad>` is
RFC 4648 base32 lowercase without padding. The `b3:` prefix is the
multihash code for BLAKE3-256 (multihash table). This display form is
multihash-compatible and round-trips with `multibase` decoders, enabling
CAR import/export and IPFS placement without an additional adapter
layer (§12 IPFS locator).

A drop's `DropId` is independent of every representation choice. The same
plaintext sealed with XChaCha20-Poly1305 then zstd-compressed and the
same plaintext stored uncompressed under no AEAD share the SAME
`DropId`. Readers MUST verify `BLAKE3(decoded_plaintext) == DropId` after
all representation inversions; this hash check is the canonical test of
identity. The AEAD tag and the EC shard hash are fast-fail optimizations
that fire before the BLAKE3 check.

#### 1.2 Image identity

`ManifestRoot` is the identity of an entire `.limni` image. It is the
Merkle root over the layer-2 metadata blob and the manifest's own
sections, as defined in §5 Merkle root construction. Two images with the
same `ManifestRoot` are byte-for-byte equivalent under this spec: they
reference the same metadata, the same slabs, and the same drops with
the same representations.

`ManifestRoot` is the only handle by which an image is identified at rest
and in transit. URLs, container IDs, content hashes, and IPFS CIDs all
reference images by `ManifestRoot`. Distribution channels (HTTP, S3,
IPFS, P2P) MAY add their own identifiers for caching and deduplication,
but those identifiers MUST resolve to `ManifestRoot` and the resolved
image MUST byte-equal what `ManifestRoot` names.

#### 1.3 Representation plane separation

Codec, AEAD, erasure coding, and locator placement are collectively the
*representation plane*. They never appear in the identity plane.
Concrete consequences:

- A drop MAY have multiple `Representation` rows in the manifest's slab
  index, each pointing at a different `(slab_id, offset, len)` tuple
  with a different `(codec, aead, ec)` triple. All such rows decode to
  the same plaintext and therefore share `DropId` (§1.1).
- A reader's `DropSource` (architecture §I2) takes a `DropId` and a
  `Representation` as input. The reader selects one representation
  (locator racing per architecture §I9), decodes through it, and
  verifies `BLAKE3(decoded_plaintext) == DropId`. The hash verification
  is canonical; representation metadata is not.
- Deepening (representation upgrade from epilimnion to hypolimnion) is
  a strict representation-plane operation: it APPENDS a new
  `Representation` row and never mutates an existing one or changes any
  `DropId`.
- Locator mirroring is likewise representation-plane: a slab may have
  multiple locator entries pointing at the same `(codec, aead, ec)`
  triple stored at different physical locations.

#### 1.4 Determinism

Every byte of a `.limni` image MUST be a deterministic function of the
input tree and the manifest's recorded parameters, with two exceptions:
the image creation timestamp (in the history section, §5.9) and the
image key (in the crypto params section, §5.5). Concretely:

- **Chunking**: same input bytes + same FastCDC parameters
  (min/avg/max, gear table ID) ⇒ same set of drops. Writers MUST NOT
  introduce any nondeterministic source (random padding, system time)
  in the chunking pipeline.
- **Classification**: same drop head bytes + same classifier registry ⇒
  same class assignment.
- **Codec encode**: same input bytes + same codec id + same level ⇒
  same output bytes. The conformance harness in `02-conformance`
  verifies this.
- **Slab packing**: same drop sequence + same packing parameters ⇒
  same slabs in the same order. Slice traversal order during build is
  spec-fixed (lexicographic by slice key, §6.2).
- **Manifest assembly**: same inputs ⇒ same sections, same section
  hashes, same `ManifestRoot`.

A nondeterministic byte found in CI (same input, two runs, different
`ManifestRoot`) is a bug class tracked under `02-conformance`. The
conformance harness runs the build pipeline twice and asserts byte
equality of the resulting manifests.

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

## Part II — The three layers

### 3. Drop store (Layer 1)

This section is normative. The drop store is the bulk-data layer of a
`.limni` image; everything else is metadata.

#### 3.1 Slab format

A slab is a contiguous byte object in the drop store. It is the unit
of locator addressing for Layer 1. Slab structure:

```
+---------------------------------+
| SlabHeader (fixed, §3.2)        |   ← magic, version, slab_id, length, EC, crypto hint
+---------------------------------+
| DropRecord[0]   (§3.3)          |
| ...                             |
+---------------------------------+
| DropRecord[n-1]                 |
+---------------------------------+
| SolidWindow[0]  (§3.4)          |   ← concatenated compressed bytes for drops 0..k
| ...                             |
+---------------------------------+
| SolidWindow[m-1]                |
+---------------------------------+
| ECShards (optional, §3.5)       |   ← k+m shards when Reed-Solomon is enabled
+---------------------------------+
```

Slabs MUST be between 4 MiB and 64 MiB inclusive, unless the manifest's
parameters section overrides (see §5.4). Slabs MUST NOT span locator
entries; one slab maps to one primary locator plus optional mirror
entries.

#### 3.2 SlabHeader

The slab header is a fixed-size prefix at offset 0:

| Field | Type | Notes |
|---|---|---|
| magic | 4 bytes | `LIM1` (0x4C 0x49 0x4D 0x31) |
| format_version | u16 LE | bump-incompatible layout changes |
| slab_id | SlabId (40 bytes) | §2.2 |
| total_length | u64 LE | bytes including header |
| ec_descriptor | u8 | 0 = no EC; 1..N = EC id (matches §16); 255 = extended descriptor (post-v1) |
| crypto_hint | u8 | 0 = plaintext; N = AEAD id (§10); the actual key lives in the manifest (§5.5) |

`magic` lets readers reject misidentified bytes before parsing the
rest. `format_version` is per-layer; see §17 versioning.
`ec_descriptor = 0` means no erasure coding; readers MUST NOT attempt
reconstruction. `ec_descriptor = 255` means the extended descriptor
follows (post-v1).

The header is followed by drop records, then solid windows, then
optional EC shards. Field order is fixed.

#### 3.3 Drop records

Each drop within a slab has a `DropRecord` describing where its bytes
live. A `DropRecord` is:

| Field | Type | Notes |
|---|---|---|
| drop_id | DropId (32 bytes) | §1.1 |
| plaintext_len | u32 LE | bytes after codec/AEAD inverse; readers size the destination buffer from this |
| representation | 3 bytes | §2.2 (codec, aead, ec); `aead = 0` and `ec = 0` allowed when the slab is plaintext |
| solid_window_index | u8 | index of the solid window containing this drop's bytes |
| offset_in_window | u32 LE | byte offset within the decompressed solid window |
| len_in_window | u32 LE | byte length within the decompressed solid window |

The triple `(solid_window_index, offset_in_window, len_in_window)` tells
the reader where to find this drop's plaintext inside the decompressed
solid window. `solid_window_index` is needed because a slab MAY have
multiple solid windows (one per class group, see §20.1 decision).

`DropRecord`s are packed contiguously in declaration order. The total
record count equals the count of drops whose representations point at
this slab; readers MAY validate this against the manifest's slab index.

#### 3.4 Solid windows

A solid window is the codec output for one or more consecutive drops.
Per §20.1: per-slab solid windows only (cross-slab class groups deferred
to `solid-blocks-v2`).

A solid window's boundaries are recorded by:
- The `DropRecord`s that reference it (their `solid_window_index` field).
- The next window's first drop record, or end-of-slab.

The codec used is `representation.codec`. When `codec = 0` (store), the
window's bytes are the literal concatenation of drop plaintexts, in
order, with no framing; readers MUST consume exactly `len_in_window`
bytes per drop.

When `codec != 0`, the window is compressed; readers decompress once
per window and then index by `(offset_in_window, len_in_window)`.

#### 3.5 EC shards (optional)

When `SlabHeader.ec_descriptor != 0` and `!= 255`, the slab's last `m`
blocks of size `ceil(|payload| / k)` are Reed-Solomon parity shards
(`k+m` total). See §16 and `02-algorithms.md §7` for the exact shard
field layout and reconstruction procedure.

The manifest's slab index (§5.4) MUST list the shard records for any
EC-enabled slab; readers resolve drops through the locator layer and
reconstruct only when fewer than `k` data shards are available.

### 4. Filesystem metadata (Layer 2)

This section is normative for the semantics of every metadata field.
The binary encoding is FlatBuffers and is owned by
[01-flatbuffers-schema]; this section is the source of truth for what
each field MEANS.

#### 4.1 Inode

An inode represents a filesystem object. Every entry in the directory
tree references an inode. Inode fields:

| Field | Type | Notes |
|---|---|---|
| number | u64 LE | inode number, unique within the image |
| mode | u32 LE | POSIX mode bits (file type + permissions) |
| uid | u32 LE | owner user ID |
| gid | u32 LE | owner group ID |
| mtime_ns | u64 LE | modification time, nanoseconds since Unix epoch |
| ctime_ns | u64 LE | inode change time |
| atime_ns | u64 LE | access time (may be absent in some images; see §4.5) |
| nlink | u32 LE | hard-link count |
| xattrs | repeated XAttr | per-inode extended attributes (§4.4) |
| content_handle | ContentHandle | the actual data (§4.3) |

`number` is the inode's wire identifier and is stable across rebuilds
of the same logical filesystem (when content-derived; see §8 for how
inode numbers are assigned in delta chains).

`mode` MUST encode a valid POSIX file type (`S_IFREG`, `S_IFDIR`,
`S_IFLNK`, etc.). Readers MUST reject unknown file types with
`Unsupported(path)`.

`mtime_ns`, `ctime_ns`, `atime_ns` are nanoseconds since Unix epoch.
Readers MUST NOT assume second granularity; the value is exact.

#### 4.2 Directory representation

A directory inode's `content_handle` is a directory tree node.
[01-flatbuffers-schema] chooses between a B-tree layout and a flat
sorted array plus offset index. The spec leaves this open until the
schema task commits; whichever layout is chosen, the byte order of
entries within a directory MUST be lexicographic by name so that range
reads and diff walks are deterministic.

Directory entries: `(name: string, inode_number: u64 LE, entry_type: u8)`.
`entry_type` is `0x01 = file`, `0x02 = directory`, `0x03 = symlink`,
`0x04 = special` (block, char, fifo, socket).

#### 4.3 Content handles and slice maps

A regular file inode's `content_handle` is a slice map:

| Field | Type | Notes |
|---|---|---|
| inline_data | `Option<[u8]>` | present only for files at or below the inline-data threshold (§2.1) |
| slices | repeated `(byte_range, slice_id)` | ordered list of slices |

For files above the threshold, `inline_data` is absent and `slices`
contains the file's chunks in byte order. A `slice_id` resolves to a
`(byte_range_in_slice, DropId)` mapping in the manifest's slice map
table; readers follow the chain to fetch drops.

Symlink inodes: `content_handle` is a `redirect` field containing the
target path string (no slice map). Symlink targets MUST be relative
paths; absolute paths are resolved relative to the image's mount root.

Special inodes (block, char, fifo, socket): `content_handle` carries
the device number or pipe identifier; readers handle as appropriate.

#### 4.4 xattrs

Extended attributes are `(namespace, key, value)` triples:

| Field | Type | Notes |
|---|---|---|
| namespace | u8 | `0x00 = user`, `0x01 = trusted`, `0x02 = system`, `0x03 = security` |
| key | string | UTF-8, no NUL bytes |
| value | bytes | arbitrary |

Readers MUST enforce the namespace policy of the host OS (e.g.,
`security.SELinux` requires privilege).

#### 4.5 atime omission

Some images omit `atime` to save space (write-heavy workloads). When
omitted, readers MUST treat `atime` reads as returning `mtime_ns` and
MUST NOT report the omission as an error.

#### 4.6 Per-class records (Seine)

For each drop in the image, the metadata layer records the Seine
classification (§13) so deepening (§8.4) is reproducible:

| Field | Type | Notes |
|---|---|---|
| drop_id | DropId | §1.1 |
| class_id | u8 | Seine class registry §13 |

These records live alongside the slice map and are part of the
layer-2 metadata blob (so the Merkle root in §5.10 commits to them).

### 5. Manifest (Layer 3)

This section is normative. The manifest is the small signed head of
the image; the `ManifestRoot` is the image's identity (§1.2).

#### 5.1 Magic + format header

The first 16 bytes of every `.limni` manifest:

| Field | Type | Notes |
|---|---|---|
| magic | 4 bytes | `LMFS` (0x4C 0x4D 0x46 0x53) |
| drop_store_version | u16 LE | §17 |
| metadata_version | u16 LE | §17 |
| manifest_version | u16 LE | §17 |
| reserved | 4 bytes | MUST be zero |

`magic` lets readers reject misidentified bytes. The three version
fields are independent (per-layer versioning, §17).

#### 5.2 Feature flags

A list of `(flag_id, required: bool)` tuples. Each flag references the
feature-flag registry (§14). Required flags unknown to the reader MUST
cause `UnsupportedFeature(flag)`; optional flags unknown to the reader
MUST be ignored (§18).

#### 5.3 Metadata reference

`H(metadata) = BLAKE3(layer_2_metadata_blob)` plus a locator for the
metadata blob. Locator formats are in §12. The Merkle root (§5.10)
commits to `H(metadata)` directly so an attacker cannot swap the
metadata blob without invalidating the root.

For small images (metadata blob ≤ 1 MiB by default; the schema task
[01-flatbuffers-schema] may adjust this), the locator MAY be `inline`
and the metadata blob embedded in this section.

#### 5.4 Slab index

A list of `(slab_id, locator_entries[])` tuples — one entry per slab
referenced by this image. `slab_id` is the `SlabId` (§2.2); locator
entries are per §12.

The slab index MUST contain every slab referenced by every drop in
the metadata layer. Conversely, slabs in the index MUST be referenced
by at least one drop (otherwise they are garbage; turnover GC handles
this — §8.3).

#### 5.5 Crypto params

| Field | Type | Notes |
|---|---|---|
| image_key_id | u8 | `0x00` = no encryption; 1..N = AEAD id (§10) |
| image_key | 32 bytes | present iff `image_key_id != 0` |
| nonce_params | NonceParams | nonce derivation (per `02-algorithms.md §5`) |
| ad_params | AdParams | associated-data construction (per `02-algorithms.md §5`) |
| recipients | repeated `HPKEEnvelope` | per-recipient key wrapping (§15) |
| signature_bundle | `Option<SignatureBundle>` | optional sigstore signature over the Merkle root |

When `image_key_id = 0`, the image is plaintext (slab crypto hint field
MUST also be 0; readers reject mismatches).

`recipients` is the per-recipient HPKE envelopes wrapping the
`image_key`. Adding a recipient appends an envelope (image key
unchanged, drops untouched). Removing a recipient drops an envelope
(residual access is a stated threat-model property — re-key = new
image version).

#### 5.6 EC params (optional)

When present, the image uses Reed-Solomon EC (§16) by default:

| Field | Type | Notes |
|---|---|---|
| k | u8 | data shards per slab (typical: 4) |
| m | u8 | parity shards per slab (typical: 2) |
| polynomial | u16 LE | GF(2^8) polynomial; default `0x011D` |
| per_slab_overrides | map<SlabId, (k, m)> | optional |

#### 5.7 DMS policy (optional)

When present, the image carries a Dead Man's Switch / key escrow
record. v0.1 supports Shamir k-of-n only (§21.2 — time-lock deferred):

| Field | Type | Notes |
|---|---|---|
| scheme | u8 | `0x00 = Shamir` |
| k | u8 | shares required to reconstruct |
| n | u8 | total shares |
| shares | repeated `ShareRecord` | `ShareRecord = (custodian_id, share_data)` |
| reconstruction_hint | `Option<string>` | optional human-readable note |

#### 5.8 Delta linkage

When this image is a delta (§8.1), this section carries:

| Field | Type | Notes |
|---|---|---|
| base_root | ManifestRoot | the parent image's `ManifestRoot` |
| tree_ops | repeated `TreeOp` | `Add | Remove | Replace` (§20.2 — no first-class `Rename` op in v0.1) |

When the image is not a delta, this section is absent. Readers detect
deltas by section presence.

#### 5.9 History

An append-only log of operations applied to derive this image from its
inputs:

| Field | Type | Notes |
|---|---|---|
| op | u8 | `0x01 = build`, `0x02 = delta`, `0x03 = flatten`, `0x04 = turnover`, `0x05 = deepen` |
| timestamp | u64 LE | nanoseconds since Unix epoch (or 0 in deterministic mode, §1.4) |
| inputs | repeated `ManifestRoot` | the inputs to this operation |
| params | bytes | operation-specific parameters (opaque to readers) |

`history` entries are deterministic except for `timestamp`. The
conformance harness verifies determinism by running the build pipeline
twice and asserting equality of everything except `timestamp`.

#### 5.10 Merkle root

```
ManifestRoot = BLAKE3(
  "limnifs/v1"
  || H(metadata)
  || H(format_header)
  || H(feature_flags)
  || H(metadata_reference)
  || H(slab_index)
  || H(crypto_params)
  || H(ec_params)
  || H(dms_policy)
  || H(delta_linkage)
  || H(history)
)
```

The domain separator `limnifs/v1` prevents cross-protocol confusion.
Flat construction (not deep tree) so each section is individually
verifiable. The metadata hash appears directly so swapping the metadata
blob invalidates the root. Sections that are absent in this manifest
(e.g., `delta_linkage` for a non-delta image) are treated as
zero-length empty strings in their slot.
---

## Part III — Addressing, overlays, derivation

### 6. Two-level addressing

This section is normative. Two-level addressing is the load-bearing
resolution rule: it determines how a reader turns a byte-range
request on a logical file into a concrete byte sequence.

#### 6.1 Resolution chain

A read of byte range `[start, end)` of a slice proceeds in three steps:

1. **Slice → drops**: consult the slice map for the slice's containing
   inode (§4.3). Each entry is `(byte_range, slice_id)`. The range
   `[start, end)` is intersected with each slice's `byte_range`;
   matching slices are mapped to their `DropId`s via the manifest's
   slice map table.
2. **Drop → slab extent**: consult the slab index (§5.4). For each
   `DropId`, find the slab record that contains it; the extent is
   `(slab_id, offset_in_window, len_in_window, representation)` from
   the `DropRecord` (§3.3) plus the slab record's locator entries.
3. **Slab → bytes**: via the locator layer (§12), fetch the slab. The
   reader decompresses the relevant solid window(s) to obtain the drop
   plaintexts. The `DropSource` post-condition (§1.1) verifies
   `BLAKE3(decoded_plaintext) == DropId`.

The three steps MUST run in this order. Implementations MUST NOT skip
step 2 by guessing at slab offsets; the slab index is the only source
of truth for `DropId → slab extent`.

#### 6.2 Range read invariants

For a slice spanning drops `{d_0, d_1, …, d_n}` with each `d_i` having
`plaintext_len = L_i`, the slice's byte range `[start, end)` resolves
to:

- For each `d_i`, compute the intersection
  `[start, end) ∩ [prefix_sum_i, prefix_sum_{i+1})`.
- If non-empty, the partial drop `d_i[start_in_drop, end_in_drop]` is
  read.
- The reader MUST return bytes in slice order; partial drops MUST be
  reassembled into the requested range.

The reader MUST NOT inflate a full slab outside recorded solid blocks
(§20.1). Specifically:

- The reader MUST fetch only the slab extents overlapping the slice
  range; reading the entire slab is a bug.
- The reader MUST decompress only the solid windows overlapping the
  slice range; decompressing unrelated windows is a bug.

The `SlabRef` field order pinned in §2.2 (`SlabId`, `offset`, `len`,
`Representation`, then locator count + entries) is the only field
order readers accept. A `SlabRef` with fields in any other order is a
malformed `SlabRef`; readers reject with `Corrupt { offset, reason }`.

### 7. Overlay chains

This section is normative. An overlay chain is a sequence of manifests
where each non-base manifest declares `base_root` (§5.8) pointing to
its parent.

#### 7.1 Chain construction and resolution

A chain is built by applying delta operations (§8.1) to a base manifest.
Resolution walks the chain from the latest manifest back to the base,
merging tree operations in reverse:

1. Start at the leaf manifest (the one being opened).
2. For each manifest in the chain (leaf → base), apply its `tree_ops`
   to the resolved tree view.
3. Stop at the base (the manifest without `delta_linkage`).

Tree operations (`Add | Remove | Replace`, §5.8) are applied in the
order they appear in each manifest's `tree_ops` list. The resolved tree
is the union of the base tree and all applied operations, with later
operations in the chain overriding earlier ones.

The reader MUST apply tree operations deterministically: same chain
order, same operation order, ⇒ same resolved tree. The conformance
harness verifies this.

#### 7.2 Depth limits

The format imposes no depth limit on overlay chains. Depth is bounded
by reader policy; a default of `overlay_max_depth = 64` is recommended
(the value is set per-reader, not in the spec). When a reader's policy
is exceeded, it MUST return `Policy { rule: "overlay_max_depth" }`.

`overlay_max_depth` is not part of the wire format; it is a reader
configuration. Authors MAY choose chains deeper than any reader's
limit, but those chains are unreadable by that reader. Conformance
vectors include chain depths 1, 2, 5, and 64.

#### 7.3 Cycle detection

Manifests MUST NOT contain cycles in their `base_root` chain. A cycle
is when manifest A's chain (via `base_root`) returns to A.

Readers detect cycles by maintaining a set of seen `ManifestRoot`s while
walking the chain. If a `ManifestRoot` is encountered twice in a single
walk, the reader MUST reject with `Policy { rule: "overlay-cycle" }`.

#### 7.4 Meromictic state

A meromictic chain (§2.1) is one that is never flattened or turned
over. Meromictic chains are valid long-term state — readers MUST NOT
require flattening. Flatten (§8.2) is an optimization for performance,
not a correctness requirement.

A reader's read path through a meromictic chain has per-read cost
O(chain_depth) metadata, not O(data movement). This is the performance
profile that makes meromictic chains practical for long-tail use cases
(e.g., cumulative security patches).

### 8. Derivation operations

This section is normative. A derivation operation takes one or more
manifests and produces a new manifest. Each operation appends to the
new manifest's history (§5.9).

#### 8.1 Delta

A delta derives a child manifest from a base manifest:

```
delta(base, next) =
  Δtree_ops = diff(base.tree, next.tree)         // 02-algorithms §8
  Δdrops    = drops added by next (via 04-writer)
  manifest  = base manifest
              with delta_linkage { base_root: base.root, tree_ops: Δtree_ops }
              with slab_index extended with new drops' slab refs
              with history append { op: delta, inputs: [base.root] }
              with new Merkle root
```

The `tree_ops` are `Add | Remove | Replace` (no first-class `Rename` per
§20.2). The writer pipeline (04) computes the tree diff and produces
new drops for added/replaced content.

The diff algorithm MUST be deterministic — same `(base, next)` inputs
produce the same `tree_ops`. The conformance harness verifies this by
diffing twice and asserting equality.

#### 8.2 Flatten

Flatten derives a composite manifest from a chain, with zero drop-store
I/O:

```
flatten(chain [m_0 (base) … m_n]) =
  tree       = resolve(chain)                    // §7
  metadata   = tree.metadata
  slab_index = union of slab refs across chain
  manifest   = build_manifest(metadata, slab_index)
                with history append { op: flatten, inputs: chain.roots }
                with new Merkle root
```

Post-condition: `resolve(flatten(chain))` is byte-identical to
`resolve(chain)`. The conformance harness verifies this — a flatten
that produces a different resolved tree is a spec violation.

Performance: O(total metadata of chain). No data movement. Flatten is
the cheap operation; use it when defragmentation isn't needed.

#### 8.3 Turnover

Turnover derives a standalone manifest, repacking all drops into new
slabs:

```
turnover(chain, sink) =
  tree = resolve(chain)                          // §7
  for each slice in tree:
    for each drop_id in slice.slice_map:
      bytes    = read_drop(drop_id)              // from chain via 03
      new_rep  = chosen_rep(chain)               // per policy
      sink.put_drop(drop_id, new_rep, bytes)
  manifest = build_manifest(tree.metadata, sink.slabs)
              with history append { op: turnover, inputs: chain.roots }
              with new Merkle root
```

GC is implicit and exact: anything unreachable from the resolved tree
is never copied. Mark-and-sweep == copy-the-live-set. The conformance
harness verifies that every drop in the new manifest is reachable and
every reachable drop is in the new manifest.

Post-conditions:

- Standalone (no external slab references).
- Byte-identical tree to `resolve(chain)`.
- Cancel-safe: abandoning before `sink.commit` leaves no reader-visible
  state.

Performance: O(live data) I/O — the only tier that moves bytes.

#### 8.4 Deepen

Deepen re-encodes drops to better representations (e.g., LZ4 → zstd
for hypolimnion-class drops). Deepen is a strict representation-plane
operation:

```
deepen(image, policy) =
  for each drop in image where representation should change:
    bytes     = read_drop(drop_id)               // from image via 03
    new_rep   = policy.new_representation(drop, current_rep)
    sink.put_drop(drop_id, new_rep, bytes)       // append, not replace
  manifest = image manifest
              with slab_index extended with new representations
              with history append { op: deepen, inputs: [image.root] }
              with new Merkle root
```

Identity invariant: deepen APPENDS new representation rows. It MUST NOT
mutate or delete existing representation rows. The original `DropId`
relationships (§1.1) are preserved — the same plaintext still addresses
identically, with one or more representation rows pointing at it.

Performance: depends on policy. Per-drop cost is `O(plaintext_len)`
encode time for the new codec.

## Part IV — Variation points (registries as data)

### 9. Registry format

This section is normative. Every algorithm, codec, locator scheme,
classifier class, and feature flag is recorded as a registry row in a
data file, never as a `match` arm in core code. This is the OCP backbone.

#### 9.1 Data files

Each registry is a single data file under `registries/<name>.toml`
(TOML preferred for human-edited files; JSON for machine-generated
ones). The schema for every registry row:

| Field | Type | Notes |
|---|---|---|
| id | per-registry | numeric ID; width (u8 / u16) is the registry's choice |
| mnemonic | string | short name; kebab-case |
| reference | string | RFC number, paper title+URL, or spec section |
| status | enum | `standard` \| `experimental` \| `deprecated` |
| params | object | algorithm-specific parameters; optional |
| description | string | human-readable note |

`id` MUST be unique within its registry. `id = 0` is reserved (means
"none" in registries that allow a no-op value, e.g., AEAD `0x00` for
plaintext). `id` MUST NOT be reused across the three states
(experimental → standard → deprecated).

`status` progression is monotonic: `experimental → standard →
deprecated`. A `deprecated` row MUST NOT be removed (deprecation is
preserved so historical images remain readable).

#### 9.2 Adding a row

Adding a new algorithm = adding a row to the registry + regenerating
bindings. NO consumer code changes. The conformance harness verifies
this property by checking that no source file outside `01-format-spec-v01`
and `01-flatbuffers-schema` references a registry ID by literal number.

Worked example: a new AEAD "Ascon-128" with id `0x05` requires only
a new row in `registries/aead.toml` and a regenerated Rust binding in
`limnifs-format`. The `05-crypto` crate's registry module picks up the
new algorithm at link time without any code change.

#### 9.3 Generation

The registry tables in this spec (§10–14) are generated from the TOML
data files. Codegen produces:

- Rust: `limnifs_format::registry::aead::Id(0x01)` style enum variants.
- Python: `limnifs_py.format.registry.aead.Id(0x01)` enum variants.
- Markdown: §10–14 of this spec (the human-readable form).

CI diff-gates the generated output against the committed code. Any
drift fails the build.

### 10. AEAD registry

Initial contents (frozen at v0.1):

| ID | Algorithm | Status | Reference |
|---|---|---|---|
| `0x00` | (none) | standard | plaintext; no AEAD applied |
| `0x01` | XChaCha20-Poly1305 | standard (mandatory baseline) | RFC 8439 |
| `0x02` | AES-128-OCB | standard | RFC 7253 |
| `0x03` | AES-256-GCM | standard | NIST SP 800-38D |
| `0x04` | Ascon-128a | standard (embedded) | NIST LW winner 2023 |

`0x01` is mandatory: every reader MUST support XChaCha20-Poly1305 to
claim spec v0.1 conformance. Other rows are mandatory if the reader
claims support for the named status.

Deterministic nonce construction (per `02-algorithms.md §5`):
`nonce = HKDF-BLAKE3(image_key, info = slab_id ‖ u64le(drop_index))[0..24]`.
The nonce derivation is a property of the registry row's parameters
schema, not a per-reader choice.

Associated-data construction: `ad = manifest_root ‖ slab_id ‖
u64le(drop_index)`. Same derivation rule.

### 11. Codec registry

Initial contents:

| ID | Codec | Status | Reference |
|---|---|---|---|
| `0x00` | store | standard (mandatory) | identity; no transform |
| `0x01` | lz4 | standard (mandatory) | LZ4 spec |
| `0x02` | zstd | standard | RFC 8478 |
| `0x03` | lzma | experimental | LZMA spec |
| `0x04` | brotli | experimental | RFC 7932 |

`0x00` (store) and `0x01` (lz4) are mandatory for v0.1 conformance. The
writer pipeline may use any registered codec; readers MUST support all
mandatory codecs and MAY support experimental ones.

Codec determinism requirement: same input bytes + same codec id + same
level parameter ⇒ same output bytes. The conformance harness verifies
this for every registered codec. Codecs that fail this test are
rejected from the registry.

### 12. Locator scheme registry

Initial contents:

| Scheme | Status | Reference | Notes |
|---|---|---|---|
| `file:` | standard (mandatory) | file path | local filesystem; no range streaming |
| `http:` | standard | RFC 7230 | range requests (§10.1 of design) |
| `https:` | standard | RFC 7230 | range requests; same wire format as `http:` over TLS |
| `s3:` | standard | AWS S3 API | multipart + conditional PUT for atomic write |
| `ipfs:` | standard | IPFS | CAR interop; drop names are multihash-compatible |
| `limni-p2p:` | experimental | design §10.1 | peer-to-peer locator |

Locator-entry wire format:

```
locator_entry = scheme ":" scheme_specific_part
```

`scheme_specific_part` is the scheme's documented URI/IRI form. For
`file:`, this is an absolute path. For `http(s):`, this is a URL with
optional range. For `s3:`, this is `bucket/key?region=...`. For `ipfs:`,
this is a CID. For `limni-p2p:`, this is a peer address plus content
hash.

`file:` is mandatory for v0.1 conformance. Locator racing (per
architecture §I9): when multiple entries exist for one slab, the
locator layer fetches from all in parallel; first successful bytes
win; lying locators are demoted via `Integrity` propagation.

### 13. Classifier class registry

Seine classes (per `02-algorithms.md §4`):

| ID | Class | Status | Detection rule |
|---|---|---|---|
| `0x00` | binary | standard (fallback) | entropy8(head 4 KiB) < 7.99 AND not sparse |
| `0x01` | already-compressed | standard | magic ∈ {gzip, xz, zstd, lzma, zip, png, jpg, mp4, ...} OR entropy8 ≥ 7.99 |
| `0x02` | sparse | standard | nul_ratio ≥ 0.99 over full drop |
| `0x03` | text/code | standard | printable_ratio ≥ 0.95 AND nul_ratio ≈ 0 |
| `0x04` | media | standard | magic ∈ {wav, flac, raw image, ...} |

`0x00` (binary) is the fallback — every drop MUST classify to one
class, and binary is the default when no rule fires.

Classification affects *ratio only*. Any misclassification must still
round-trip (the conformance vector for class mapping verifies
round-trip on a fixed input set).

### 14. Feature-flag registry

Flag IDs (numeric, u16):

| Flag | ID | Status | Required? | Reference |
|---|---|---|---|---|
| EC | `0x0001` | standard | optional | §16 |
| DMS | `0x0002` | standard | optional | §15 |
| `file:` locator | `0x0010` | standard | optional | §12 |
| `http:` locator | `0x0011` | standard | optional | §12 |
| `https:` locator | `0x0012` | standard | optional | §12 |
| `s3:` locator | `0x0013` | standard | optional | §12 |
| `ipfs:` locator | `0x0014` | standard | optional | §12 |
| `zstd` codec | `0x0020` | standard | optional | §11 |
| `lzma` codec | `0x0021` | experimental | optional | §11 |
| `brotli` codec | `0x0022` | experimental | optional | §11 |
| `solid-blocks-v2` | `0x0100` | experimental | optional | §20.1 (deferred) |
| `rename-ops` | `0x0101` | experimental | optional | §20.2 (deferred) |
| `dms-time-lock` | `0x0102` | experimental | optional | §21.2 (deferred) |

Flag ID ranges:

- `0x0000` reserved (no flag).
- `0x0001–0x00FF` standard flags.
- `0x0100–0x01FF` experimental flags.
- `0x0200–0xFFFF` reserved for future use.

`required` flags unknown to the reader cause `UnsupportedFeature(flag)`
(§18.1). `optional` flags unknown to the reader are ignored (§18.2).

## Part V — Cryptography and redundancy (references)

### 15. Cryptography

References the algorithm specs. The wire-format-relevant invariants:

- Image key: 32 B random per image, wrapped per recipient via HPKE
  (X25519-DHKEM, HKDF-SHA256, ChaChaPoly).
- AEAD application: nonce = `HKDF-BLAKE3(image_key, info = slab_id ‖ u64le(drop_index))[0..24]`;
  AD = `manifest_root ‖ slab_id ‖ u64le(drop_index)`.
- Deterministic nonces are safe because data is immutable (slab, index
  tuples are unique by construction).
- Signature bundle is sigstore-compatible; signing is optional, BLAKE3
  integrity is always present.
- Full algorithm specs live in component `05-crypto`.

### 16. Erasure coding

- Reed-Solomon over GF(2^8), per-slab k+m; polynomial table fixed.
- Shard records carry their own BLAKE3 hashes; reconstruction MUST
  verify against the slab hash before yielding.
- Representation variant; identity unchanged.
- Full algorithm specs live in component `07-erasure-coding`.

---

## Part VI — Versioning and conformance

### 17. Versioning policy

- Per-layer version numbers: `drop_store_version`, `metadata_version`,
  `manifest_version`.
- A reader declaring spec vX.Y MUST accept any image whose every layer
  version is ≤ the reader's declared maximum for that layer.
- Feature flags negotiate post-version capabilities.
- Deprecation: tombstone flag + new field/path; IDs and field offsets are
  NEVER reused.

### 18. Unknown-flag policy

- Required flag unknown to reader: fail with `UnsupportedFeature(flag)`.
- Optional flag unknown to reader: ignore; reader behavior stays defined.
- Each flag's required/optional classification lives in the registry (§14).

### 19. Conformance

- Conformance vectors in component `02-conformance` exercise every
  boundary condition stated in this spec.
- A reader claiming spec v0.1 conformance MUST pass all v0.1 vectors.
- The Python reference reader (`limnifs/limnifs-py`) is the spec-sufficiency
  oracle: if the Rust reader and the Python reader agree on all vectors,
  the spec is unambiguous; if they disagree, the spec has a gap.

---

## Part VII — Resolved and deferred design questions

### 20. Resolved in v0.1

#### 20.1 Solid-block boundaries (design §16.1)

**Decision:** per-slab solid windows with explicit boundaries in metadata.

A solid block is a codec output that concatenates consecutive drops
within a single slab into one compressed stream. The slab index records
each drop's byte range within the *decompressed* solid window. A slab
MAY contain multiple solid windows; each window's boundaries are
recorded.

**Rationale:** keeps slab addressing clean — every drop's bytes resolve
to `(slab_id, offset, len)` within one slab. Cross-slab class groups
(the alternative) would require a new addressing indirection and a
multi-slab inflation path; they are deferred to a `solid-blocks-v2`
feature flag. Adding that flag later does not break v0.1 readers
(they treat it as required-unknown and reject, or optional-unknown and
fall back to per-slab windows).

**Cost:** small-files ratio within a single slab. Cross-slab class
locality is the deeper optimization deferred to v2.

#### 20.2 Rename semantics in delta manifests (design §16.4)

**Decision:** v0.1 has NO first-class `Rename` op. The delta builder
MUST compile detected renames to `Remove(from)` + `Add(to)` pairs.

**Rationale:** the v0.1 delta op set is `Add / Remove / Replace` only.
Rename detection still happens at delta build time (the writer walks
add/remove pairs and matches identical content id sets); only the
encoding choice changes. The decision is reversible: a future
`rename-ops` feature flag can add `Rename(from, to, content_id)`
without breaking v0.1 readers (the flag is optional-unknown to them;
they fall back to compiled Remove+Add form).

**Cost:** delta manifests for rename-heavy workloads are slightly larger
than first-class rename would be (two ops + duplicated content id vs.
one op). Recoverable in v2 if Phase 1+ benchmarks demand.

### 21. Deferred (owned by other components)

#### 21.1 FastCDC parameters and minimum drop size (design §16.2)

**Owner:** `04-writer-pipeline`. The manifest's parameter section
carries chunking parameters (min/avg/max sizes, gear table ID); the
format itself is parameter-agnostic. 04 chooses specific defaults with
benchmarks; once chosen, this spec records them as a referenced
parameter set in §3 (drop records).

#### 21.2 Time-lock puzzle calibration (design §16.3)

**Owner:** `05-crypto`. v0.1 ships Shamir k-of-n escrow only. Time-lock
puzzle code MUST NOT land until this spec defines parameter selection
and a Wesolowski/Pietrzak proof-of-elapsed-time scheme. The
`dms-time-lock` feature flag is registered in §14 as optional and
default-off; flipping it to standard requires a spec amendment.

---

## Part VIII — Worked examples (stubs)

Each example walks through the bytes of a small image at v0.1. The full
byte-level walkthrough is added when the corresponding sections above
have complete prose AND the matching conformance vector exists. Outline
summaries only:

### 22.1 Single uncompressed image

One regular file, single drop, store codec, no crypto, no EC. Exercises:
manifest assembly, Merkle root, slab index with one entry, inode with
one slice-map entry.

### 22.2 Delta chain of depth 2

Base image (22.1) plus a delta adding one file and replacing another.
Exercises: `base_root` linkage, tree ops (Add, Replace), history
append, chain resolution.

### 22.3 Encrypted image (single recipient)

Like 22.1 with AEAD id = 0x01 (XChaCha20-Poly1305), one HPKE envelope.
Exercises: crypto params section, image key wrap, deterministic nonce
derivation, AD construction.

### 22.4 Erasure-coded image (k=4, m=2)

Like 22.1 with RS over the single slab. Exercises: EC params section,
shard records, reconstruction-and-verify path.

---

## Appendices

### A. References

- Architecture docs: `00-architecture/{00-overview,01-interfaces,02-algorithms,03-comparison}.md`.
- Design doc: `docs/superpowers/specs/2026-07-28-limnifs-design.md`.
- RFCs: 2119 (normative language), 7515 (HPKE dependencies), 9180 (HPKE),
  7253 (OCB), 5869 (HKDF), 8439 (ChaCha20-Poly1305).
- BLAKE3 specification (the BLAKE3 team).
- sigstore: ` Fulcio / Rekor / Cosign ` ecosystem specs (reference).

### B. Change log

- **v0.1-draft-outline** (this PR): section structure, decisions in §20
  (solid blocks per-slab; renames compiled). Open questions deferred to
  04 (CDC parameters) and 05 (DMS calibration) per §21.

---

## What is NOT in this outline (deliberate)

To keep the outline reviewable, the following are deferred to follow-up
PRs and are NOT in this document:

- The actual `.fbs` schema files (`schema/*.fbs`) — added by
  [01-flatbuffers-schema].
- The registry data files (`registries/*.toml`) — added by
  [01-feature-flag-registry].
- Full prose for each section above (only structure and decisions are
  here; the prose is filled in section-by-section in follow-up PRs).
- Byte-level worked examples (§22 stubs only; expanded when the matching
  conformance vectors exist).
- Choice of directory-structure encoding (B-tree vs flat array + index)
  in §4 — pinned by [01-flatbuffers-schema].
- Inline-data threshold for very small files — added when §4 prose lands.
- The SHA-pinning of conformance vectors — added by `02-conformance`.
