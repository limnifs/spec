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
