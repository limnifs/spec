# 43 — EC params section (bit-level)

- **Source:** [§5.6 EC params (optional)](../wire-format/23-manifest.md)
- **Total width:** variable (`11 + 42 × override_count`)
- **Alignment:** immediately follows the crypto params section (§5.5)
  when both are present. Absent entirely when the image does not use
  Reed-Solomon EC.
- **Section version:** 1.

The EC params section configures Reed-Solomon erasure coding for
the image's slabs. Per spec §5.6, the section is OPTIONAL — it is
present iff the EC feature flag (`0x0001`) is declared in the
feature flags section (§5.2).

When present, the section pins the default `(k, m)` pair, the
GF(2^8) polynomial, and any per-slab overrides. Slabs inherit the
default unless they appear in the override table.

## Section layout (v1)

```
+----------------------------------------------------+
| section_version       : u8   (= 1)                 |  offset 0
+----------------------------------------------------+
| k                     : u8   (data shards)         |  offset 1
+----------------------------------------------------+
| m                     : u8   (parity shards)       |  offset 2
+----------------------------------------------------+
| polynomial            : u16 LE (GF(2^8))           |  offset 3
+----------------------------------------------------+
| override_count        : u32 LE                     |  offset 5
+----------------------------------------------------+
| overrides[]           : override_count × Override  |  offset 9
+----------------------------------------------------+
```

Total width: `9 + 42 × override_count` bytes.

## Override layout (42 bytes fixed)

Each override pins a per-slab `(k, m)` pair that differs from the
default:

```
+----------------------------------------------------+
| slab_id.ordinal : u64 LE                           |  offset 0
+----------------------------------------------------+
| slab_id.hash    : 32 bytes                         |  offset 8
+----------------------------------------------------+
| k               : u8                               |  offset 40
+----------------------------------------------------+
| m               : u8                               |  offset 41
+----------------------------------------------------+
```

Total: 42 bytes per override (matches `SlabId` width 40 + 2 bytes
for the `(k, m)` pair).

## Byte-offset table (section level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | section_version | u8 | — | currently `1` |
| 1 | 1 | k | u8 | — | data shards per slab; typical 4; MUST be ≥ 1 |
| 2 | 1 | m | u8 | — | parity shards per slab; typical 2; MUST be ≥ 1 |
| 3 | 2 | polynomial | u16 | LE | GF(2^8) generator polynomial; default `0x011D` |
| 5 | 4 | override_count | u32 | LE | number of per-slab overrides |
| 9 | `42 × override_count` | overrides | Override[] | — | packed contiguously |

## Field semantics

### section_version (offset 0, u8)

The layout version of this section. Currently `1`. Bumped on
bump-incompatible changes to the section or to the override record.

### k (offset 1, u8)

The default number of data shards per slab. Each slab's payload
(size `total_payload`) is split into `k` data shards of size
`ceil(total_payload / k)` per spec §3.5.

Constraints:
- `k >= 1` (otherwise no data shards; the slab is unreachable).
- `k <= 256` (the Reed-Solomon upper bound for GF(2^8)).
- The slab-size ceiling (default 64 MiB) divided by `k` is the
  per-shard size. For `k = 4` and 64 MiB slabs, shards are 16 MiB
  each.

### m (offset 2, u8)

The default number of parity shards per slab. Reed-Solomon
generates `m` parity shards from the `k` data shards; any `m` of
the `k + m` total shards can be lost without data loss.

Constraints:
- `m >= 0` (zero is valid — equivalent to no EC, but expressed
  inside the EC params section; readers MAY reject `m = 0` here as
  `Corrupt` since declaring EC with no parity is contradictory).
- `k + m <= 256` (GF(2^8) shard count limit).

### polynomial (offset 3..5, u16 LE)

The GF(2^8) generator polynomial used by the Reed-Solomon code.
Default `0x011D` per spec §5.6 (the canonical AES polynomial).
Readers MUST verify the polynomial matches what their RS
implementation expects; mismatched polynomials produce silently
incorrect reconstruction.

The polynomial is encoded as a u16 to leave room for future
extension to GF(2^16) (post-v1).

### override_count (offset 5..9, u32 LE)

The number of per-slab overrides that follow. Zero is valid and
common — most images use a single default `(k, m)` pair.

### overrides (offset 9.., variable)

`override_count` consecutive 42-byte override records. Each
record's `slab_id` MUST be unique within the section (no slab
gets two overrides); duplicates are `Corrupt`.

An override's `slab_id` MUST appear in the slab index section
(§5.4). An override for a non-indexed slab is `Corrupt`
(unreferenced slabs are garbage per §8.3; this cross-check runs
at the slab-walker layer).

An override's `(k, m)` MUST satisfy the same constraints as the
default (`k >= 1`, `m >= 1`, `k + m <= 256`).

## Validation rules

An EC params section is valid iff:

1. At least 9 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `k >= 1`. Otherwise: `Corrupt`.
4. `m >= 1`. Otherwise: `Corrupt` (zero parity while declaring EC
   is contradictory).
5. `k + m <= 256`. Otherwise: `Corrupt` (GF(2^8) limit).
6. `polynomial` matches a known-good value (default `0x011D`).
   Readers MAY accept additional polynomials; reject unknown with
   `UnsupportedFeature`.
7. `9 + 42 × override_count` does not overrun the manifest buffer.
8. Each override's `slab_id` is unique within the section.
9. Each override's `(k, m)` satisfies rules 3–5.

Cross-section consistency (every override references a slab in
the slab index) runs at the slab-walker layer after the slab
index is also parsed.

## Worked example

### Default-only EC params (k=4, m=2, no overrides)

A v0.1 image with EC enabled, default `(4, 2)`, AES polynomial,
no per-slab overrides:

```
01                section_version = 1
04                k = 4 data shards
02                m = 2 parity shards
1D 01             polynomial = 0x011D (LE)
00 00 00 00       override_count = 0
```

9 bytes total. Every slab inherits `(4, 2)`.

### With one per-slab override

Default `(4, 2)`, plus slab ordinal 7 overridden to `(8, 4)`:

```
01                section_version = 1
04                k = 4
02                m = 2
1D 01             polynomial = 0x011D
01 00 00 00       override_count = 1
07 00 00 00 00 00 00 00       slab_id.ordinal = 7
[32 zero bytes]               slab_id.hash (example)
08                override k = 8
04                override m = 4
```

9 + 42 = 51 bytes total. Slab 7 uses `(8, 4)`; all others use
`(4, 2)`.

## Cross-references

- [§5.6 EC params (optional)](../wire-format/23-manifest.md) —
  semantic description.
- [§3.5 EC shards (optional)](../wire-format/21-drop-store.md) —
  how `k` and `m` shape the slab's shard layout.
- [§16 Erasure coding](../crypto/16-ec.md) — Reed-Solomon over
  GF(2^8).
- [§14 Feature-flag registry](../registries/14-feature-flags.md) —
  EC flag `0x0001` that signals this section's presence.
- [30-slab-header.md](30-slab-header.md) — the slab header's
  `ec_descriptor` field that references this section's scheme.
- [39-slab-index.md](39-slab-index.md) — the slab index that
  overrides must reference.
