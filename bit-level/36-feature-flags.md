# 36 — Feature flags section (bit-level)

- **Source:** [§5.2 Feature flags](../wire-format/23-manifest.md)
- **Total width:** variable (5 + 3 × N bytes, where N is the entry count)
- **Alignment:** immediately follows the 16-byte manifest header
  (offset 16).
- **Section version:** 1 (current; per [pivot
  D3](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md)
  — every section carries its own version byte).

The feature flags section declares which optional features the image
relies on, and whether each is required. Readers apply the
unknown-flag policy ([§18](../versioning/18-unknown-flag.md)) per
entry: an unknown REQUIRED flag fails the read with
`UnsupportedFeature(flag)`; an unknown optional flag is silently
ignored.

## Section layout (v1)

```
+----------------------------------------------------+
| section_version : u8     (= 1)                     |  offset 0
+----------------------------------------------------+
| entry_count     : u32 LE (number of flag entries)  |  offset 1
+----------------------------------------------------+
| entry[0]       : 3 bytes                           |  offset 5
+----------------------------------------------------+
| entry[1]       : 3 bytes                           |  offset 8
+----------------------------------------------------+
| ...                                                |
+----------------------------------------------------+
| entry[N-1]     : 3 bytes                           |  offset 5 + 3(N-1)
+----------------------------------------------------+
```

Total width: `5 + 3 × N` bytes.

## Entry layout (3 bytes)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 2 | flag_id | u16 | LE | references [§14 Feature-flag registry](../registries/14-feature-flags.md) |
| 2 | 1 | required | u8 | — | `0x00` = optional, `0x01` = required; any other value is `Corrupt` |

Flag IDs use the ranges pinned in
[§14](../registries/14-feature-flags.md):

- `0x0000` reserved (no flag); readers SHOULD reject if encountered.
- `0x0001–0x00FF` standard flags.
- `0x0100–0x01FF` experimental flags.
- `0x0200–0xFFFF` reserved for future use.

`required` is a single byte (not a bool) so the entry stays
alignment-friendly and so future versions can extend the field
(e.g., a `required_min_version` byte) without breaking the v1 layout.

## Bit-position diagram

For a section declaring two flags — EC (required) and `https:`
locator (optional):

```
Section: 5 + 3 + 3 = 11 bytes

Offset 0    1    2    3    4    5    6    7    8    9   10
      +----+----+----+----+----+----+----+----+----+----+----+
      | 01 | 02 | 00 | 00 | 00 | 01 | 00 | 01 | 12 | 00 | 00 |
      +----+----+----+----+----+----+----+----+----+----+----+
      | vr |<-- entry_count = 2 -->|< entry 0 >|< entry 1 >|
      |    |                       | 01 00 | 01 | 12 00 | 00 |
      |    |                       | EC    |req| https |opt|
      |    |                       | flag  |   | flag  |   |
      | vr = section_version = 1
```

Each cell is one byte, shown in hexadecimal.

## Field semantics

### section_version (offset 0, u8)

The layout version of this section. Currently `1`. Bumped on
bump-incompatible changes to the section layout (entry width, count
width, addition of new entry fields, etc.). A reader that does not
recognise the section version MUST surface `UnsupportedFeature` and
stop parsing further manifest sections — there is no skip-and-resume
in v0.1 because the section sequence is fixed.

### entry_count (offset 1..5, u32 LE)

The number of feature-flag entries that follow. Bounded by the
manifest's overall size; readers MUST verify `5 + 3 × entry_count`
does not exceed the bytes available in the section. A count that
overruns the buffer is `Corrupt`.

The maximum count is constrained by registry cardinality (currently
13 entries; future registries may add more). Readers MUST NOT assume
any particular upper bound other than the buffer-overflow check.

### flag_id (per entry offset 0..2, u16 LE)

The flag identifier, referencing the
[§14 Feature-flag registry](../registries/14-feature-flags.md).
Readers consult the registry to determine whether the flag is known
and, if known, what it implies for further parsing.

### required (per entry offset 2, u8)

`0x00` = optional. The flag declares a feature the image uses; if
the reader doesn't know the flag, it ignores the entry (§18.2).

`0x01` = required. The image cannot be read correctly without
support for this flag; if the reader doesn't know the flag, it
fails the read with `UnsupportedFeature(flag_id)` (§18.1).

Any other value is `Corrupt` — readers MUST NOT silently coerce
nonzero-non-one values to "required".

## Validation rules

A feature flags section is valid iff:

1. At least 5 bytes are available (version + count).
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `5 + 3 × entry_count` does not overrun the manifest buffer.
4. Every entry's `flag_id` is in the registry's ID space (`0x0001`
   through the current maximum; `0x0000` is `Corrupt`).
5. Every entry's `required` is `0x00` or `0x01`. Other values:
   `Corrupt`.
6. No duplicate `flag_id`s within the section. Duplicates:
   `Corrupt` (a flag is either required or optional; declaring it
   twice is contradictory).

Duplicate-flag detection runs after parsing completes — readers MUST
not skip this check; it would let two entries disagree about
`required` for the same flag.

## Worked example

### Empty flags section (no flags set)

```
01 00 00 00 00
|  |<- count=0 ->|
|  section_version=1
```

5 bytes total. Reader advances 5 bytes to the next section.

### Single required flag (EC)

```
01                  section_version = 1
01 00 00 00         entry_count = 1
01 00               flag_id = 0x0001 (EC)
01                  required = true
```

8 bytes total. A reader that does not implement EC MUST fail with
`UnsupportedFeature(0x0001)`.

### All standard v0.1 flags declared optional

Twelve standard flags (excluding reserved `0x0000`), each with
`required = 0x00`:

```
01                  section_version = 1
0C 00 00 00         entry_count = 12
01 00 00            EC         optional
02 00 00            DMS        optional
10 00 00            file:      optional
11 00 00            http:      optional
12 00 00            https:     optional
13 00 00            s3:        optional
14 00 00            ipfs:      optional
20 00 00            zstd       optional
21 00 00            lzma       optional
22 00 00            brotli     optional
00 01 00            solid-blocks-v2  optional
01 01 00            rename-ops       optional
```

5 + 3 × 12 = 41 bytes total.

## Cross-references

- [§5.2 Feature flags](../wire-format/23-manifest.md) — semantic
  description.
- [§14 Feature-flag registry](../registries/14-feature-flags.md) —
  the flag IDs and their meanings.
- [§18 Unknown-flag policy](../versioning/18-unknown-flag.md) — what
  readers do with unknown flags (required vs optional).
- [§17 Versioning policy](../versioning/17-versioning.md) — what
  bumping a section version means.
- [35-manifest-header.md](35-manifest-header.md) — the header that
  precedes this section.
