# 40 — History section (bit-level)

- **Source:** [§5.9 History](../wire-format/23-manifest.md)
- **Total width:** variable (`5 + Σ entry_lengths`)
- **Alignment:** immediately follows the delta linkage section
  (§5.8) when present, otherwise follows the DMS policy (§5.7).
  In a minimal image with no optional sections, follows the slab
  index (§5.4) directly.
- **Section version:** 1.

The history section is an append-only log of operations applied to
derive this image from its inputs. Every image MUST have at least
one history entry (the `build` op that produced it). Delta, flatten,
turnover, and deepen operations append additional entries.

History entries are deterministic except for the `timestamp` field.
The conformance harness verifies determinism by running the build
pipeline twice and asserting equality of everything except
`timestamp`.

## Section layout (v1)

```
+----------------------------------------------------+
| section_version : u8   (= 1)                       |  offset 0
+----------------------------------------------------+
| entry_count     : u32 LE                           |  offset 1
+----------------------------------------------------+
| entries[]       : entry_count × HistoryEntry       |  offset 5
+----------------------------------------------------+
```

Total width: `5 + Σ entry_lengths` bytes.

## HistoryEntry layout

Each entry records one operation:

```
+----------------------------------------------------+
| op              : u8                               |  offset 0
+----------------------------------------------------+
| timestamp       : u64 LE                           |  offset 1
+----------------------------------------------------+
| input_count     : u32 LE                           |  offset 9
+----------------------------------------------------+
| inputs[]        : input_count × 32 bytes           |  offset 13
+----------------------------------------------------+
| params_len      : u32 LE                           |  variable
+----------------------------------------------------+
| params          : params_len bytes                 |  variable
+----------------------------------------------------+
```

Entry width: `13 + input_count × 32 + params_len` bytes.

## Byte-offset table (section level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | section_version | u8 | — | currently `1` |
| 1 | 4 | entry_count | u32 | LE | MUST be ≥ 1 (every image has a build entry) |
| 5 | variable | entries | HistoryEntry[] | — | packed contiguously in append order |

## Field semantics

### op (entry offset 0, u8)

The operation that produced this image version. Defined opcodes:

| Value | Name | Meaning |
|---|---|---|
| `0x01` | build | Initial construction from a directory tree |
| `0x02` | delta | Diff against a parent image (§8.1) |
| `0x03` | flatten | Metadata-only overlay collapse (§8.2) |
| `0x04` | turnover | Full repack: flatten + defrag + re-encode + GC (§8.3) |
| `0x05` | deepen | Representation-plane upgrade (epilimnion → hypolimnion) (§8.4) |
| `0x06`–`0xFE` | reserved | Future ops; readers reject with `UnsupportedFeature` |
| `0xFF` | extended | Extended op follows (post-v1); readers reject with `UnsupportedFeature` |

### timestamp (entry offset 1..9, u64 LE)

Nanoseconds since the Unix epoch. MAY be `0` in deterministic mode
(§1.4): conformance harnesses build with `timestamp = 0` and assert
that two runs produce byte-identical manifests.

### input_count (entry offset 9..13, u32 LE)

Number of `ManifestRoot` inputs to the operation. For `build`, this
is `0` (the input is the source directory tree, not another image).
For `delta`, this is `1` (the parent image). For `flatten`,
`turnover`, `deepen`, this is `1` or more (the overlay chain).

### inputs (entry offset 13.., variable)

`input_count` consecutive 32-byte `ManifestRoot` values per §1.2.
Each input references a parent image by its identity.

### params_len (variable offset, u32 LE)

Byte length of the operation-specific parameters blob. `0` when the
op has no parameters (e.g., a build with default settings). The
params are opaque to readers — they're interpreted only by the
writer that produced the entry.

### params (variable offset, params_len bytes)

Operation-specific parameters. Default ceiling `4 KiB` (per the
manifest's parameters section, when that exists). Examples:

- `build`: FastCDC parameters, classifier config, codec levels.
- `delta`: diff algorithm id, similarity threshold.
- `turnover`: target tier policy, GC aggressiveness.

The format of `params` is operation-specific and not specified at
the bit level in v0.1.

## Validation rules

A history section is valid iff:

1. At least 5 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `entry_count >= 1`. An image with no history is `Corrupt` (every
   image has at least the build entry).
4. Each entry's `op` is in `{0x01, 0x02, 0x03, 0x04, 0x05}`. Other
   values: `UnsupportedFeature` (or `Corrupt` for `0xFF`).
5. Each entry's `params_len` does not exceed the configured
   ceiling (default 4 KiB). Otherwise: `Corrupt`.
6. `5 + Σ entry_lengths` does not overrun the bytes available in
   the containing manifest buffer.

Cross-entry consistency (e.g., "a `delta` entry's `inputs[0]` must
match the manifest's `delta_linkage.base_root`") runs at a higher
layer once the delta linkage section (§5.8) is also parsed.

## Worked example

### Single build entry, deterministic (timestamp = 0)

A build operation with no inputs and no params:

```
01                              section_version = 1
01 00 00 00                      entry_count = 1
01                              op = 0x01 (build)
00 00 00 00 00 00 00 00          timestamp = 0 (deterministic mode)
00 00 00 00                      input_count = 0
00 00 00 00                      params_len = 0
```

Total: 1 + 4 + 1 + 8 + 4 + 4 = 22 bytes.

### Delta from one parent

A delta operation with one parent input and 64 bytes of params:

```
01                              section_version = 1
01 00 00 00                      entry_count = 1
02                              op = 0x02 (delta)
00 10 12 3e 84 00 00 00          timestamp = 1735000000 sec (example)
01 00 00 00                      input_count = 1
DE AD BE EF ... 32 bytes         inputs[0] (parent ManifestRoot)
40 00 00 00                      params_len = 64
[64 bytes of op-specific params]
```

Total: 1 + 4 + 1 + 8 + 4 + 32 + 4 + 64 = 118 bytes.

### Multi-op chain: build then deepen

A small image deepened once after build:

```
01                              section_version = 1
02 00 00 00                      entry_count = 2
[HistoryEntry 0: build, t=0, inputs=0, params_len=0]
[HistoryEntry 1: deepen, t=1000, inputs=1, params_len=16]
```

The `inputs[0]` of entry 1 references the ManifestRoot of the
build's output. Verifying this requires the Merkle root (§5.10),
which commits to all section hashes including history.

## Cross-references

- [§5.9 History](../wire-format/23-manifest.md) — semantic
  description.
- [§1.2 Image identity](../concepts/11-identity.md) — `ManifestRoot`
  used as the input reference.
- [§1.4 Determinism](../concepts/11-identity.md) — `timestamp = 0`
  convention for deterministic mode.
- [§8 Derivation operations](../algorithms/42-derivations.md) — what
  each `op` actually does.
- [§5.8 Delta linkage](../wire-format/23-manifest.md) — cross-section
  consistency check for `delta` entries.
- [§5.10 Merkle root](../wire-format/23-manifest.md) — commits to
  the history section's hash.
