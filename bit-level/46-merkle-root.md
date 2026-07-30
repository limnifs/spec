# 46 — Merkle root construction (bit-level)

- **Source:** [§5.10 Merkle root](../wire-format/23-manifest.md)
- **Output width:** 32 bytes (one BLAKE3 digest).
- **Alignment:** not a stored section. The `ManifestRoot` is COMPUTED
  by the reader from the manifest's section bytes; the manifest does
  not store its own root. The computed root IS the image's identity
  (§1.2): two images with the same `ManifestRoot` are byte-for-byte
  equivalent.

The Merkle root is the canonical integrity check for a LimniFS image.
A reader that parses every section and recomputes the root gets
end-to-end integrity verification: any tampering with any byte of
any section produces a different `ManifestRoot`.

## Formula

```
ManifestRoot = BLAKE3(
    "limnifs/v1"                       // 10-byte domain separator
    || H(metadata)                     // hash of the layer-2 metadata blob
    || H(format_header)                // hash of the 16-byte manifest header
    || H(feature_flags)                // hash of the feature flags section bytes
    || H(metadata_reference)           // hash of the metadata reference section bytes
    || H(slab_index)                   // hash of the slab index section bytes
    || H(crypto_params)                // hash of crypto params section bytes (or empty)
    || H(ec_params)                    // hash of EC params section bytes (or empty)
    || H(dms_policy)                   // hash of DMS policy section bytes (or empty)
    || H(delta_linkage)                // hash of delta linkage section bytes (or empty)
    || H(history)                      // hash of the history section bytes
)
```

Where `H(x) = BLAKE3(x)` (standard 32-byte output).

## Input layout

Total input width: **330 bytes** (10-byte separator + 10 × 32-byte
hashes). The final BLAKE3 invocation reads all 330 bytes and produces
the 32-byte `ManifestRoot`.

| Offset | Width | Slot | Source |
|---|---|---|---|
| 0 | 10 | domain separator | `"limnifs/v1"` (ASCII, no NUL terminator) |
| 10 | 32 | `H(metadata)` | `metadata_reference.metadata_hash` (stored field) |
| 42 | 32 | `H(format_header)` | `BLAKE3(header_bytes[..16])` |
| 74 | 32 | `H(feature_flags)` | `BLAKE3(feature_flags_section_bytes)` |
| 106 | 32 | `H(metadata_reference)` | `BLAKE3(metadata_reference_section_bytes)` |
| 138 | 32 | `H(slab_index)` | `BLAKE3(slab_index_section_bytes)` |
| 170 | 32 | `H(crypto_params)` | `BLAKE3(crypto_params_bytes)` or `BLAKE3("")` if absent |
| 202 | 32 | `H(ec_params)` | `BLAKE3(ec_params_bytes)` or `BLAKE3("")` if absent |
| 234 | 32 | `H(dms_policy)` | `BLAKE3(dms_policy_bytes)` or `BLAKE3("")` if absent |
| 266 | 32 | `H(delta_linkage)` | `BLAKE3(delta_linkage_bytes)` or `BLAKE3("")` if absent |
| 298 | 32 | `H(history)` | `BLAKE3(history_section_bytes)` |
| 330 | (end) | — | — |

The slot order matches the formula in [§5.10](../wire-format/23-manifest.md).
Readers MUST concatenate the slots in this exact order; any permutation
produces a different `ManifestRoot` and breaks image identity.

## Field semantics

### Domain separator (offset 0..10)

The 10 ASCII bytes `"limnifs/v1"` (no NUL terminator). Prevents
cross-protocol confusion: a BLAKE3 collision with another protocol's
hash output cannot be replayed as a LimniFS `ManifestRoot`.

The separator is version-stamped (`v1`). A future incompatible
revision of the Merkle formula MUST bump the separator (e.g.,
`"limnifs/v2"`) so old readers cannot confuse new roots for old ones.

### H(metadata) (offset 10..42)

BLAKE3 of the layer-2 metadata blob (the actual filesystem metadata
content, NOT the metadata reference section bytes). The metadata
reference section stores this hash directly as its `metadata_hash`
field; readers use that stored value here.

Two manifests pointing at the same metadata blob share this slot.
The metadata blob's identity is decoupled from the manifest's
section layout — the same metadata can be referenced by many
manifests (e.g., in a delta chain where the metadata didn't change).

### H(format_header) (offset 42..74)

BLAKE3 of the 16-byte manifest header (§5.1). Includes the magic
`LMFS`, the three version fields, and the reserved bytes. A reader
that re-parses the header gets the same bytes; any tampering with
magic, versions, or reserved produces a different hash.

### H(feature_flags) (offset 74..106)

BLAKE3 of the feature flags section bytes (§5.2), starting from the
section_version byte through the last entry. The section bytes are
what the parser consumes; the hash commits to every byte including
the leading version byte.

### H(metadata_reference) (offset 106..138)

BLAKE3 of the metadata reference section bytes (§5.3). This INCLUDES
the `metadata_hash` field — so swapping the metadata hash invalidates
the root, even before the metadata blob itself is fetched.

### H(slab_index) (offset 138..170)

BLAKE3 of the slab index section bytes (§5.4). Commits to the
declared slab identifiers and locator lists.

### H(crypto_params) (offset 170..202)

BLAKE3 of the crypto params section bytes (§5.5) when present;
otherwise `BLAKE3("")` (the empty-section convention). For a v0.1
plaintext image (`image_key_id = 0`), this section is absent and
the slot is `BLAKE3("")`.

### H(ec_params) (offset 202..234)

BLAKE3 of the EC params section bytes (§5.6) when present; otherwise
`BLAKE3("")`. For a v0.1 image without Reed-Solomon EC, the section
is absent.

### H(dms_policy) (offset 234..266)

BLAKE3 of the DMS policy section bytes (§5.7) when present;
otherwise `BLAKE3("")`. For a v0.1 image without DMS key escrow,
the section is absent.

### H(delta_linkage) (offset 266..298)

BLAKE3 of the delta linkage section bytes (§5.8) when present;
otherwise `BLAKE3("")`. For a non-delta image, the section is absent.

### H(history) (offset 298..330)

BLAKE3 of the history section bytes (§5.9). Commits to the operation
log including all `timestamp` values. Two images built from identical
inputs but at different times have different `ManifestRoot`s (per §1.4
determinism: timestamp is the only nondeterministic field).

## Absent-section convention

Sections that are absent in this manifest (e.g., `delta_linkage` for
a non-delta image) are treated as zero-length empty strings in their
slot. `BLAKE3("")` is a specific 32-byte value (the empty-input
digest); readers SHOULD call `BLAKE3(&[])` directly rather than
hardcoding the constant.

The empty-section convention lets the formula stay fixed across
image variants: a non-EC image and an EC-enabled image use the same
formula; the EC-disabled image simply has `BLAKE3("")` in the EC
slot.

## Validation rules

A reader computing the Merkle root:

1. Parses every section that is present.
2. Records the raw bytes consumed by each section parser (the
   `cursor`'s position delta from section start to section end).
3. Computes `H(section_bytes)` for each section.
4. For absent sections, uses `BLAKE3(&[])`.
5. Concatenates the 10-byte domain separator + 10 × 32-byte hashes.
6. Computes `BLAKE3(concatenated)` to obtain the 32-byte
   `ManifestRoot`.

The computed root is the image's identity. There is no separate
"declared ManifestRoot" field to compare against; the root IS the
name by which the image is addressed.

## Worked example

Consider a minimal v0.1 image: plaintext, no EC, no DMS, not a
delta. Eight of the ten hash slots are populated from actual
sections; two are `BLAKE3("")`.

```
Domain separator:    6c 69 6d 6e 69 66 73 2f 76 31           "limnifs/v1"
H(metadata):         [32 bytes from metadata_reference.metadata_hash]
H(format_header):    [BLAKE3 of the 16-byte LMFS header]
H(feature_flags):    [BLAKE3 of the flags section]
H(metadata_reference): [BLAKE3 of the metadata reference section]
H(slab_index):       [BLAKE3 of the slab index section]
H(crypto_params):    af 13 49 b9 f5 f9 a1 a6 ... (BLAKE3(""))
H(ec_params):        af 13 49 b9 f5 f9 a1 a6 ... (BLAKE3(""))
H(dms_policy):       af 13 49 b9 f5 f9 a1 a6 ... (BLAKE3(""))
H(delta_linkage):    af 13 49 b9 f5 f9 a1 a6 ... (BLAKE3(""))
H(history):          [BLAKE3 of the history section]
```

The final `BLAKE3` over these 330 bytes produces the image's
`ManifestRoot`, displayed in the `b3:<base32>` form per §1.1.

## Cross-references

- [§5.10 Merkle root](../wire-format/23-manifest.md) — semantic
  formula.
- [§1.2 Image identity](../concepts/11-identity.md) — `ManifestRoot`
  is THE image handle.
- [§1.4 Determinism](../concepts/11-identity.md) — why timestamp
  matters.
- [§5.3 Metadata reference](../wire-format/23-manifest.md) — source
  of the `metadata_hash` field used as `H(metadata)`.
- [bit-level/35-manifest-header.md](35-manifest-header.md) — the
  16 bytes hashed as `H(format_header)`.
- [bit-level/36-feature-flags.md](36-feature-flags.md),
  [bit-level/38-metadata-reference.md](38-metadata-reference.md),
  [bit-level/39-slab-index.md](39-slab-index.md),
  [bit-level/40-history.md](40-history.md) — the section bytes
  hashed in the other slots.
