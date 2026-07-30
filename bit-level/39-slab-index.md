# 39 — Slab index section (bit-level)

- **Source:** [§5.4 Slab index](../wire-format/23-manifest.md)
- **Total width:** variable (`5 + Σ entry_lengths`)
- **Alignment:** immediately follows the metadata reference section
  (§5.3) in the manifest's section sequence.
- **Section version:** 1.

The slab index is the manifest's table of contents for the drop
store. One entry per slab referenced by this image. Each entry
identifies the slab by `SlabId` (§2.2) and lists the locator entries
that can fetch it.

The slab index MUST contain every slab referenced by every drop in
the metadata layer (§4.3). Conversely, slabs in the index MUST be
referenced by at least one drop (otherwise they are garbage;
turnover GC handles this — §8.3).

## Section layout (v1)

```
+----------------------------------------------------+
| section_version : u8   (= 1)                       |  offset 0
+----------------------------------------------------+
| entry_count     : u32 LE                           |  offset 1
+----------------------------------------------------+
| entries[]       : entry_count × SlabIndexEntry     |  offset 5
+----------------------------------------------------+
```

Total width: `5 + Σ entry_lengths` bytes.

## SlabIndexEntry layout

Each entry identifies one slab plus its locators:

```
+----------------------------------------------------+
| slab_id.ordinal : u64 LE                           |  offset 0
+----------------------------------------------------+
| slab_id.hash    : 32 bytes                         |  offset 8
+----------------------------------------------------+
| locator_count   : u32 LE (>= 1)                    |  offset 40
+----------------------------------------------------+
| locator_entries : locator_count × LocatorEntry     |  offset 44
+----------------------------------------------------+
```

Entry width: `40 + 4 + Σ locator_entry_lengths` bytes.

The `SlabId` is encoded exactly as in
[30-slab-header.md](30-slab-header.md) bytes 6..46: ordinal first
(u64 LE), then 32-byte hash.

## Byte-offset table (section level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | section_version | u8 | — | currently `1` |
| 1 | 4 | entry_count | u32 | LE | number of slab entries that follow |
| 5 | variable | entries | SlabIndexEntry[] | — | packed contiguously in declaration order |

## Field semantics

### section_version (offset 0, u8)

The layout version of this section. Currently `1`. Bumped on
bump-incompatible changes to the section or to `SlabIndexEntry`.

### entry_count (offset 1..5, u32 LE)

Number of `SlabIndexEntry` records that follow. MUST be `>= 1` for
any non-empty image (a manifest with no slabs is degenerate; readers
reject with `Corrupt` unless the metadata layer also declares no
drops — that cross-check runs at the slab-walker layer).

### entries (offset 5.., variable)

`entry_count` consecutive `SlabIndexEntry` records in declaration
order. Declaration order SHOULD match slab ordinal order (smallest
ordinal first) for determinism (§1.4); readers do not require this
ordering but writers SHOULD emit it.

## SlabIndexEntry field semantics

### slab_id (offset 0..40, SlabId)

The slab's identifier per §2.2: u64 LE ordinal + 32-byte BLAKE3 hash.
The ordinal distinguishes slabs that hash to the same value within
one image; the hash is the content-addressed part.

Duplicate `slab_id`s within one slab index are `Corrupt` — a slab
appears at most once.

### locator_count (offset 40..44, u32 LE)

Number of locator entries that follow for this slab. MUST be `>= 1`
— a slab with no locators is unreachable and `Corrupt`.

Locator racing (§I9): when `locator_count > 1`, readers fetch from
all in parallel; first successful bytes win; lying locators are
demoted via `Integrity` propagation.

### locator_entries (offset 44.., variable)

`locator_count` consecutive `LocatorEntry` records per
[37-locator-entry.md](37-locator-entry.md). Each entry is a
length-prefixed URI in the `scheme ":" scheme_specific_part` form.

A slab MAY have multiple locator entries with the SAME scheme (e.g.,
two `https:` mirrors) or with DIFFERENT schemes (e.g., `file:` for
local cache + `https:` for fallback). Readers race all entries.

## Validation rules

A slab index section is valid iff:

1. At least 5 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `entry_count >= 1` (unless the image is empty; cross-checked at
   the slab-walker layer against the metadata).
4. Each entry's `slab_id` is unique within the section. Duplicates:
   `Corrupt`.
5. Each entry's `locator_count >= 1`. Otherwise: `Corrupt`.
6. Each locator entry is individually valid per
   [37-locator-entry.md](37-locator-entry.md) rule set.
7. `5 + Σ entry_lengths` does not overrun the bytes available in
   the containing manifest buffer.

Cross-section consistency (rule: every slab referenced by a drop
appears in the index; rule: every slab in the index is referenced
by at least one drop) runs at the slab-walker layer after the
metadata layer is parsed.

## Worked example

### Single slab, single locator

A tiny image with one slab at `file:///var/lib/limnifs/slab-0.bin`:

```
01                              section_version = 1
01 00 00 00                      entry_count = 1
00 00 00 00 00 00 00 00          slab_id.ordinal = 0
[32 zero bytes]                  slab_id.hash
01 00 00 00                      locator_count = 1
1F 00 00 00                      locator length = 31
[31 bytes "file:///var/lib/limnifs/slab-0.bin"]
```

Total: 1 + 4 + (8 + 32 + 4 + 4 + 31) = 84 bytes.

### Two slabs, mirrored

Two slabs each mirrored to `file:` and `https:`:

```
01                              section_version = 1
02 00 00 00                      entry_count = 2
[SlabId 0]                       slab 0
02 00 00 00                      locator_count = 2
[file: locator]
[https: locator]
[SlabId 1]                       slab 1
02 00 00 00                      locator_count = 2
[file: locator]
[https: locator]
```

Reader can race `file:` vs `https:` per slab for resilience.

## Cross-references

- [§5.4 Slab index](../wire-format/23-manifest.md) — semantic
  description.
- [§2.2 Semantic types](../01-glossary.md) — `SlabId` definition.
- [§3.2 SlabHeader](../wire-format/21-drop-store.md) — what the
  slab bytes contain once fetched.
- [37-locator-entry.md](37-locator-entry.md) — locator entry layout.
- [38-metadata-reference.md](38-metadata-reference.md) — the
  section that precedes the slab index.
- [§8.3 Turnover](../algorithms/42-derivations.md) — garbage
  collection of unreferenced slabs.
