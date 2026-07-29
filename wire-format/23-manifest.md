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
| reserved | 6 bytes | MUST be zero |

`magic` lets readers reject misidentified bytes. The three version
fields are independent (per-layer versioning, §17). The reserved field
pads the header to a 16-byte aligned fixed width (power-of-two, common
header convention); readers MUST reject non-zero reserved bytes as a
`Corrupt` error so future spec amendments can repurpose the field
without silent misinterpretation by older readers.

#### 5.2 Feature flags

A list of `(flag_id, required: bool)` tuples. Each flag references the
feature-flag registry (§14). Required flags unknown to the reader MUST
cause `UnsupportedFeature(flag)`; optional flags unknown to the reader
MUST be ignored (§18).

Byte-level layout: see [bit-level/36-feature-flags.md](../bit-level/36-feature-flags.md).
Section version 1: `[version: u8][entry_count: u32 LE][entry × N]`
where each entry is `[flag_id: u16 LE][required: u8]` (3 bytes).
Section begins at offset 16 (immediately after the manifest header).

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
