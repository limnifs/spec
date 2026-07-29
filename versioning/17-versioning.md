### 17. Versioning policy

This section is normative.

#### 17.1 Per-layer versions

Three independent version numbers in the manifest header (§5.1):

- `drop_store_version`
- `metadata_version`
- `manifest_version`

Each starts at `1` for spec v0.1. Bump-incompatible changes (wire
format breaking) increment the corresponding version. The spec
reserves version `0` for "version not set" — readers MUST reject
version `0`.

#### 17.2 Compatibility rules

A reader declaring support for spec vX.Y MUST accept any image whose:

- `drop_store_version ≤ drop_store_max` (per-reader limit)
- `metadata_version ≤ metadata_max`
- `manifest_version ≤ manifest_max`

If any version exceeds the reader's limit, the reader returns
`UnsupportedVersion { layer, version, max }`.

#### 17.3 Feature flags vs versions

Feature flags negotiate POST-version capabilities. A reader
implementing spec v1.0 might not implement the `solid-blocks-v2`
flag; it sees the flag as `optional`-unknown and falls back to
per-slab windows (§20.1).

A version bump means a wire-format breaking change. A new feature
flag means a new optional capability. The two mechanisms are
independent — never conflate them.

#### 17.4 Deprecation

IDs and field offsets are NEVER reused. Deprecation = new field +
tombstone flag. Historical images with deprecated fields MUST remain
readable.
