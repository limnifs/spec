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
