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
