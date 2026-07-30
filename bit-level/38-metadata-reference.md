# 38 — Metadata reference section (bit-level)

- **Source:** [§5.3 Metadata reference](../wire-format/23-manifest.md)
- **Total width:** variable (`37 + locator_bytes + inline_metadata_len`)
- **Alignment:** immediately follows the feature flags section (§5.2)
  in the manifest's section sequence.
- **Section version:** 1 (per [pivot
  D3](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md)).

The metadata reference tells the reader where to fetch the
filesystem metadata blob (Layer 2) and commits to that blob's
BLAKE3 hash. The Merkle root (§5.10) includes `H(metadata)` directly
so swapping the metadata blob invalidates the root.

For small images (≤ 1 MiB by default), the metadata blob MAY be
inlined into this section. For larger images, the blob is fetched
via one or more locator entries.

## Section layout (v1)

```
+----------------------------------------------------+
| section_version     : u8   (= 1)                   |  offset 0
+----------------------------------------------------+
| metadata_hash       : 32 bytes (BLAKE3)            |  offset 1
+----------------------------------------------------+
| locator_count       : u32 LE                       |  offset 33
+----------------------------------------------------+
| locator_entries[]   : locator_count × LocatorEntry |  offset 37
+----------------------------------------------------+
| inline_metadata_len : u32 LE                       |  variable
+----------------------------------------------------+
| inline_metadata     : inline_metadata_len bytes    |  variable
+----------------------------------------------------+
```

Total width: `37 + Σ locator_entry_lengths + 4 + inline_metadata_len` bytes.

`LocatorEntry` is `[length: u32 LE][uri bytes]` per
[37-locator-entry.md](37-locator-entry.md).

## Field semantics

### section_version (offset 0, u8)

The layout version of this section. Currently `1`. Bumped on
bump-incompatible changes to the section layout.

### metadata_hash (offset 1..33, 32 bytes)

`H(metadata) = BLAKE3(layer_2_metadata_blob)`. Readers MUST verify
that the fetched (or inlined) metadata blob hashes to exactly this
value; mismatch is `Corrupt`.

The hash commits to the metadata blob's bytes — independent of
codec, AEAD, or representation choices. Two images with the same
metadata blob have the same `metadata_hash`.

### locator_count (offset 33..37, u32 LE)

Number of locator entries that follow. MAY be `0` when
`inline_metadata_len > 0`. Otherwise MUST be `>= 1` — a metadata
reference with no locators and no inline blob is unreachable and
`Corrupt`.

Locator racing (§I9): when `locator_count > 1`, readers fetch from
all in parallel; first successful bytes win.

### locator_entries (offset 37.., variable)

`locator_count` consecutive `LocatorEntry` records per
[37-locator-entry.md](37-locator-entry.md). Each entry is a
length-prefixed URI in the `scheme ":" scheme_specific_part` form.

When the metadata blob is inlined, `locator_count` MAY be `0` or
MAY include alternative non-inline locators (e.g., a cache hit).
Readers prefer the inline blob when present.

### inline_metadata_len (variable offset, u32 LE)

Byte length of the inlined metadata blob that follows. `0` when
the metadata is not inlined; positive when it is. Default ceiling
`1 MiB` (configurable via the manifest's parameters section, when
that section exists in a later spec revision).

### inline_metadata (variable offset, inline_metadata_len bytes)

The metadata blob itself, byte-for-byte what `metadata_hash`
commits to. Readers verify `BLAKE3(inline_metadata) == metadata_hash`
before parsing the metadata's contents.

## Validation rules

A metadata reference section is valid iff:

1. At least 37 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `locator_count` and `inline_metadata_len` together provide at
   least one source for the metadata blob:
   - `locator_count >= 1` OR `inline_metadata_len > 0`. Otherwise:
     `Corrupt { reason: "metadata unreachable" }`.
4. Each locator entry is individually valid per
   [37-locator-entry.md](37-locator-entry.md) rule set.
5. `inline_metadata_len <= configured_ceiling` (default 1 MiB).
   Otherwise: `Corrupt`.
6. If `inline_metadata_len > 0`, the inlined bytes hash to
   `metadata_hash`. Otherwise: `Corrupt`. (This rule runs after
   the section is parsed; the parser proper only checks the
   structural rules 1–5.)

## Worked example

### External metadata, single file locator

A small image whose metadata blob lives at
`file:///var/lib/limnifs/metadata.bin`:

```
01                                                  section_version = 1
DE AD BE EF ... 32 bytes total                       metadata_hash
01 00 00 00                                          locator_count = 1
1F 00 00 00                                          locator entry length = 31
66 69 6C 65 3A 2F 2F 2F 76 61 72 ... 31 bytes        "file:///var/lib/..."
00 00 00 00                                          inline_metadata_len = 0
                                                     (no inline bytes)
```

Total: 1 + 32 + 4 + 4 + 31 + 4 = 76 bytes.

### Inlined metadata

A small image with the metadata blob inlined (1024 bytes):

```
01                                                  section_version = 1
12 34 56 ... 32 bytes total                          metadata_hash
00 00 00 00                                          locator_count = 0
00 04 00 00                                          inline_metadata_len = 1024
[1024 bytes of metadata]                             inline_metadata
```

Total: 1 + 32 + 4 + 4 + 1024 = 1065 bytes. Reader verifies
`BLAKE3(inline_metadata) == metadata_hash`.

### Mirrored metadata with inline fallback

A medium image mirrored to HTTPS and S3, with an inline fallback:

```
01                                                  section_version = 1
[32 bytes metadata_hash]
02 00 00 00                                          locator_count = 2
[https locator entry]
[s3 locator entry]
00 10 00 00                                          inline_metadata_len = 4096
[4096 bytes inline_metadata]
```

Reader races the two locators; if both fail, falls back to inline.

## Cross-references

- [§5.3 Metadata reference](../wire-format/23-manifest.md) —
  semantic description.
- [37-locator-entry.md](37-locator-entry.md) — locator entry layout.
- [§5.10 Merkle root](../wire-format/23-manifest.md) — how
  `metadata_hash` participates in the root.
- [§I2 (architecture)](https://github.com/limnifs/limnifs/blob/main/TODO.impl/00-architecture/00-overview.md)
  — drop source / locator racing.
- [§14 Feature-flag registry](../registries/14-feature-flags.md) —
  flags that govern optional sections (EC §5.6, DMS §5.7, etc.).
