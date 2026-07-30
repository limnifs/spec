# Layer 3 — Bit-level layout

This directory is the **bit-level layer** of the LimniFS specification. It
is the source of truth a parser implementer reaches for when they need
to know exactly which byte goes where. Each file pins the wire layout
of one fixed-width type or section, down to the byte offset and bit
position.

Layer 3 is normative. Where Layer 2 (`wire-format/`) says "the slab
header contains a magic field", Layer 3 says "bytes 0..4 of every slab
are `0x4C 0x49 0x4D 0x31`". A reader implementing from Layer 3 alone
must produce a conformant parser.

## How to read Layer 3

Each file follows the same shape:

1. **Heading** — names the type and links back to its Layer 2 section.
2. **Total width** — exact size in bytes (or "variable" with the
   length-prefix rule).
3. **Byte-offset table** — one row per field, with offset, width, name,
   type, endianness, and notes. Field order is fixed.
4. **Bit-position diagram** — ASCII diagram of the byte stream with
   every field labelled, for visual cross-checking.
5. **Validation rules** — what readers MUST reject (bad magic, nonzero
   reserved, truncated input, etc.).
6. **Worked example** — a small concrete bytestring annotated
   field-by-field.

ASCII art is the primary notation; formal RFC-style byte layout is used
where it removes ambiguity. Cross-references are markdown links to the
owning Layer 2 section and to the relevant registry.

## File index

| File | Type | Source |
|---|---|---|
| [30-slab-header.md](30-slab-header.md) | 56-byte slab header (§3.2) | `wire-format/21-drop-store.md` |
| [31-drop-record.md](31-drop-record.md) | 48-byte drop record (§3.3) | `wire-format/21-drop-store.md` |
| [32-representation.md](32-representation.md) | 3-byte `Representation` triple (§2.2) | `01-glossary.md` |
| [35-manifest-header.md](35-manifest-header.md) | 16-byte manifest header (§5.1) | `wire-format/23-manifest.md` |
| [36-feature-flags.md](36-feature-flags.md) | feature flags section (§5.2) | `wire-format/23-manifest.md` |
| [37-locator-entry.md](37-locator-entry.md) | length-prefixed locator URI (§12) | `registries/12-locator.md` |
| [38-metadata-reference.md](38-metadata-reference.md) | metadata reference section (§5.3) | `wire-format/23-manifest.md` |
| [39-slab-index.md](39-slab-index.md) | slab index section (§5.4) | `wire-format/23-manifest.md` |
| [40-history.md](40-history.md) | history section (§5.9) | `wire-format/23-manifest.md` |
| [43-ec-params.md](43-ec-params.md) | EC params section (§5.6) | `wire-format/23-manifest.md` |
| [44-dms-policy.md](44-dms-policy.md) | DMS policy section (§5.7) | `wire-format/23-manifest.md` |
| [45-delta-linkage.md](45-delta-linkage.md) | delta linkage section (§5.8) | `wire-format/23-manifest.md` |
| [46-merkle-root.md](46-merkle-root.md) | Merkle root construction (§5.10) | `wire-format/23-manifest.md` |
| 33-inode.md | inode record (§4.1) | `wire-format/22-metadata.md` |
| 34-merkle-btree-node.md | directory B-tree node | `wire-format/22-metadata.md` |
| 42-crypto-params.md | crypto params section (§5.5) | `wire-format/23-manifest.md` |
| 43-ec-params.md | EC params section (§5.6) | `wire-format/23-manifest.md` |
| 44-dms-policy.md | DMS policy section (§5.7) | `wire-format/23-manifest.md` |
| 45-delta-linkage.md | delta linkage section (§5.8) | `wire-format/23-manifest.md` |

Files marked in the index but not yet present are tracked by the spec
restructure plan (`TODO.impl/01-spec/01-spec-restructure-plan.md`).
