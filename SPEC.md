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

Specifies the load-bearing addressing rule:

- Slice → drop resolution via the inode's slice map.
- Drop → slab extent resolution via the slab index:
  `(slab_id, offset, len, representation)`.
- Range reads: slice byte range → drop subset → slab extent subset.
  MUST NOT inflate a full slab outside recorded solid blocks (§20.1).

### 7. Overlay chains

Specifies delta-chain resolution:

- A manifest MAY declare a `base_root`; resolution walks the chain.
- Chain depth is unbounded in format; bounded by reader policy (default
  to be set by 03-core-reader, recorded as `overlay_max_depth`).
- Cycle detection: manifests include a `chain_depth` counter; readers
  MUST reject cycles with `Policy { rule: "overlay-cycle" }`.
- Meromictic chains are valid long-term state — readers MUST NOT require
  flattening.

### 8. Derivation operations

Specifies the manifest-recorded operations (the derivations from
`00-overview §5`). Each appends to the manifest's history:

- 8.1 **Delta**: `base_root` + tree ops list (Add / Remove / Replace at
  inode granularity) + new drops via the writer pipeline. Rename encoding
  per §20.2.
- 8.2 **Flatten** (06): composite manifest from a chain; drops
  re-referenced, not re-encoded. Post-condition: `resolve(flatten(chain))`
  is byte-identical to `resolve(chain)`.
- 8.3 **Turnover** (06+04): standalone manifest with all drops repacked;
  history records `op: turnover`.
- 8.4 **Deepen** (04): new representation rows only; identity unchanged;
  history records `op: deepen`.

---

## Part IV — Variation points (registries as data)

### 9. Registry format

Specifies the format of every registry. Each registry is a single data
file under `registries/`. Rows have: numeric ID (stable, never reused),
mnemonic, reference (RFC/paper), parameter schema, status
(standard / experimental / deprecated). Adding a row + regenerating
bindings MUST be the only step required to introduce a new algorithm —
this is the OCP enforcement.

### 10. AEAD registry

Initial contents (from design §9, frozen at v0.1):

| ID | Algorithm | Status |
|---|---|---|
| 0x01 | XChaCha20-Poly1305 | standard (mandatory baseline) |
| 0x02 | AES-128-OCB | standard |
| 0x03 | AES-256-GCM | standard |
| 0x04 | Ascon-128a | standard (embedded readers) |

### 11. Codec registry

Initial contents:

| ID | Algorithm | Status |
|---|---|---|
| 0x00 | store (identity) | standard |
| 0x01 | lz4 | standard |
| 0x02 | zstd | standard |
| 0x03 | lzma | experimental (Phase 1+) |
| 0x04 | brotli | experimental (Phase 1+) |

### 12. Locator scheme registry

Initial contents: `file:`, `http(s):`, `s3:`, `ipfs:`. `limni-p2p:` is
experimental. Each scheme's locator-entry encoding is specified in this
section.

### 13. Classifier class registry

Seine classes: `already-compressed`, `sparse`, `text/code`, `media`,
`binary` (fallback). Spec restates the constraint from
[02-algorithms §4]: classification affects representation choice only;
any class MUST round-trip.

### 14. Feature-flag registry

Flag IDs include: EC, DMS, each locator scheme, post-v1 codecs,
overlay-depth policy, `solid-blocks-v2` (deferred, §20.1),
`rename-ops` (deferred, §20.2), `dms-time-lock` (deferred, §21.2).
Each flag is classified required or optional (§18).

---

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
