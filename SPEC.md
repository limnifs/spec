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

Specifies the bulk-data layer:

- Slab header: magic, slab format version, `SlabId`, slab length, optional
  EC descriptor, optional crypto hint (AEAD id only — keys live in the
  manifest).
- Drop records: `(drop_id, plaintext_len, representation, offset, len)` —
  `offset`/`len` are within the slab's post-codec byte stream.
- Solid blocks: see decision §20.1 — per-slab solid windows with explicit
  boundaries; codec output may concatenate consecutive drops within a slab;
  the slab index carries each drop's byte range within the decompressed
  solid window.
- Reed-Solomon shards (optional, per-slab): shard records with their own
  BLAKE3 hashes and locator entries. Representation variant; identity
  unchanged.

### 4. Filesystem metadata (Layer 2)

Specifies the POSIX-ish metadata layer. Wire format is FlatBuffers; the
schema lives in `schema/fs.fbs` and is added by [01-flatbuffers-schema].
This section specifies the *semantics* of every field:

- Inode: number, mode, uid, gid, mtime/ctime/atime, nlink, xattrs, content
  handle (slice + slice_map for regular files; redirect target for
  symlinks; inline data for very small files — see §20 placeholder).
- Directory representation: structure deferred to the schema task (B-tree
  vs flat array + index). Spec will pin one once [01-flatbuffers-schema]
  chooses.
- Slice map: per-inode ordered list of `(byte_range, DropId)` tuples.
- xattrs: per-inode key-value list; namespace per xattr.
- Per-class records: Seine class ID per drop, used by the writer pipeline
  and recorded so deepening is reproducible (determinism, §1).

### 5. Manifest (Layer 3)

Specifies the small signed head of the image. Sections, in order:

1. Magic + format header (per-layer version numbers).
2. Feature flags section.
3. Metadata reference: `H(metadata)` (the layer-2 metadata blob's hash)
   plus a locator for the metadata blob (inline for small images; URL or
   CID for detached images). The Merkle root commits to `H(metadata)`
   directly (item 10) so this section exists to *locate* the blob, not to
   re-hash it.
4. Slab index (every slab referenced by this image, with locator entries).
5. Crypto params (image key wrapped per recipient; AEAD id; nonce
   derivation parameters).
6. EC params (optional, per-image defaults; per-slab overrides in slab index).
7. DMS policy (optional; Shamir escrow only in v0.1, see §21.2).
8. Delta linkage (`base_root` if this is a delta; tree ops list — §8.1).
9. History (append-only operation log).
10. Merkle root:
    `BLAKE3("limnifs/v1" || H(metadata) || H(section_1) || … || H(section_9))`.
    Domain separator prevents cross-protocol confusion; flat construction
    (not deep tree) so each section is individually verifiable. The
    metadata hash appears directly in the hash list so an attacker cannot
    swap the metadata blob without invalidating the root.

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
