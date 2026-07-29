### 13. Classifier class registry

Seine classes (per `02-algorithms.md §4`):

| ID | Class | Status | Detection rule |
|---|---|---|---|
| `0x00` | binary | standard (fallback) | entropy8(head 4 KiB) < 7.99 AND not sparse |
| `0x01` | already-compressed | standard | magic ∈ {gzip, xz, zstd, lzma, zip, png, jpg, mp4, ...} OR entropy8 ≥ 7.99 |
| `0x02` | sparse | standard | nul_ratio ≥ 0.99 over full drop |
| `0x03` | text/code | standard | printable_ratio ≥ 0.95 AND nul_ratio ≈ 0 |
| `0x04` | media | standard | magic ∈ {wav, flac, raw image, ...} |

`0x00` (binary) is the fallback — every drop MUST classify to one
class, and binary is the default when no rule fires.

Classification affects *ratio only*. Any misclassification must still
round-trip (the conformance vector for class mapping verifies
round-trip on a fixed input set).
