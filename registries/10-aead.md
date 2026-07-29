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
