# 35 — Manifest header (bit-level)

- **Source:** [§5.1 Magic + format header](../wire-format/23-manifest.md)
- **Total width:** 16 bytes (fixed)
- **Alignment:** 16-byte aligned (power-of-two fixed header).

The manifest header is the first 16 bytes of every `.lim` image
manifest. Readers validate the header before any other parse step. A
header that fails validation terminates parsing with a structured
`CoreError` — no further bytes are read.

## Byte-offset table

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 4 | magic | bytes | — | `LMFS` = `0x4C 0x4D 0x46 0x53` |
| 4 | 2 | drop_store_version | u16 | LE | drop store layer version (§17) |
| 6 | 2 | metadata_version | u16 | LE | metadata layer version (§17) |
| 8 | 2 | manifest_version | u16 | LE | manifest layer version (§17) |
| 10 | 6 | reserved | bytes | — | MUST be zero; readers reject nonzero |

Total: 4 + 2 + 2 + 2 + 6 = **16 bytes**.

The three version fields are independent (per-layer versioning per
[§17](../versioning/17-versioning.md)). A reader that supports manifest
version N but encounters N+1 SHOULD surface `UnsupportedFeature` rather
than guess at the layout — versions are bumped on
bump-incompatible layout changes.

## Bit-position diagram

```
Byte  0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
     | 4C | 4D | 46 | 53 | vv | vv | vv | vv | vv | vv | 00 | 00 | 00 | 00 | 00 | 00 |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
     |<- magic 'LMFS ->|< drop_store >|< metadata  >|< manifest  >|<------ reserved ----->|
     |                   u16 LE         u16 LE         u16 LE         6 zero bytes        |
```

Each cell is one byte, shown in hexadecimal. `vv` denotes a version
byte (low byte of the u16 LE; the high byte follows immediately).

## Field semantics

### magic (bytes 0..4)

The byte sequence `0x4C 0x4D 0x46 0x53` (ASCII `LMFS`). Lets readers
reject misidentified bytes before any further parse step. The bytes
spell `LMFS` in ASCII for grepability and to match the file-extension
mnemonic (`.lim` → `LimniFS` → `LMFS`).

Readers MUST reject any other magic with `CoreError::BadMagic { found }`
where `found` is the actual 4-byte sequence encountered.

### drop_store_version (bytes 4..6, u16 LE)

The drop store layer version. Currently `1`. Bumped on
bump-incompatible changes to the slab header (§3.2), drop record
(§3.3), solid window layout (§3.4), or EC shard layout (§3.5).

### metadata_version (bytes 6..8, u16 LE)

The metadata layer version. Currently `1`. Bumped on
bump-incompatible changes to the inode record (§4.1), directory
representation (§4.2), slice map (§4.3), or the deterministic Merkle
B-tree node layout.

### manifest_version (bytes 8..10, u16 LE)

The manifest layer version. Currently `1`. Bumped on
bump-incompatible changes to any section layout in §5.2–§5.9 or to
the Merkle root formula in §5.10.

### reserved (bytes 10..16, 6 bytes)

MUST be zero in v0.1. The width reconciles the section's "first 16
bytes" framing with the field table (4 + 2 + 2 + 2 = 10 used bytes;
6 reserved bytes complete the 16-byte total). The 16-byte total gives
power-of-two alignment for the header.

Readers MUST reject nonzero reserved bytes with
`CoreError::Corrupt { reason: "reserved bytes 10..16 must be zero, found [...]" }`.
This rejection rule means a future spec amendment can repurpose the
reserved field (e.g., for a fourth version field, or a flag byte)
without silent misinterpretation by older readers — those readers
will reject the new layout loudly.

## Validation rules

A header is valid iff:

1. The input is at least 16 bytes long. Shorter inputs return
   `CoreError::TooShort { have, need: 16 }`.
2. `magic == [0x4C, 0x4D, 0x46, 0x53]`. Otherwise
   `CoreError::BadMagic { found }`.
3. `reserved == [0; 6]`. Otherwise `CoreError::Corrupt { reason }`.

Version validity (e.g., "is manifest_version 1 supported?") is a
HIGHER-layer concern, handled by feature-flag policy (§18) and the
registry data (§14). The header parser only checks structural
invariants; it does not enforce version policy.

Extra bytes after offset 16 are ignored by the header parser — they
belong to §5.2 (feature flags), which begins at byte 16.

## Worked example

A v1.0.0 manifest header (all three layers at version 1):

```
4C 4D 46 53    01 00          01 00          01 00          00 00 00 00 00 00
|<- magic ->|  | drop_store |  | metadata  |  | manifest  |  |<- reserved ->|
               version = 1     version = 1     version = 1     all zero (OK)
```

A header with a corrupted reserved byte (byte 13 = `0x01`):

```
4C 4D 46 53    01 00          01 00          01 00          00 00 00 01 00 00
|<- magic ->|  | drop_store |  | metadata  |  | manifest  |  |<- reserved ->|
               version = 1     version = 1     version = 1     nonzero (REJECT)
                                                                -> Corrupt
```

## Cross-references

- [§5.1 Magic + format header](../wire-format/23-manifest.md) —
  semantic description.
- [§17 Versioning policy](../versioning/17-versioning.md) — what
  bumping a version means.
- [§18 Unknown-flag policy](../versioning/18-unknown-flag.md) — how
  readers handle versions they don't know.
- [§14 Feature-flag registry](../registries/14-feature-flags.md) — the
  flag table referenced by §5.2.
