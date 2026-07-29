### 18. Unknown-flag policy

This section is normative.

#### 18.1 Required flag unknown

When a reader encounters a required flag (§14) whose ID it does not
recognize, it MUST return `UnsupportedFeature(flag_id)` and refuse to
open the image. The flag's classification (required vs optional) is
recorded in the registry; readers consult it on every open.

#### 18.2 Optional flag unknown

When a reader encounters an optional flag whose ID it does not
recognize, it MUST ignore the flag and proceed. The reader's behavior
is defined by the spec as if the flag were absent — flags describe
extensions, not invariants. An unrecognized optional flag MAY be
logged for diagnostics but MUST NOT fail the open.

#### 18.3 Unknown registry row

When a reader encounters an unknown ID in any registry (AEAD, codec,
locator scheme, classifier, feature flag), the behavior depends on
the registry:

- **AEAD**: image cannot be opened (data is opaque without the
  algorithm).
- **Codec**: image cannot be opened.
- **Locator scheme**: slab is unreadable (other slabs may be).
- **Classifier**: ignored (the classifier is write-side metadata only).
- **Feature flag**: per §18.1 / §18.2.

A reader that lists its supported registry IDs in its build
configuration (a generated Rust enum or Python dataclass) cannot
encounter "unknown" rows — it fails to compile. A reader that
interprets rows dynamically (e.g., a forward-compatible in-memory
parser) follows the rules above.
