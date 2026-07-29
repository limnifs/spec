# Deprecated: FlatBuffers schema files

**Status:** DEPRECATED 2026-07-29. Files kept per the project's
never-delete rule.

## What's deprecated

All `.fbs` files in this directory:

- `types.fbs` — semantic types (DropId, SlabId, ManifestRoot,
  Representation, Tier, Hash).
- `manifest.fbs` — manifest sections (MagicHeader, FeatureFlag,
  MetadataReference, CryptoParams, etc.).

These were drafted in session 5 as FlatBuffers schemas for the
LimniFS wire format.

## Why deprecated

The LimniFS wire format pivot (user-approved 2026-07-29; see
[2026-07-29-wire-format-pivot.md](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md))
drops FlatBuffers in favor of a custom binary format owned by
LimniFS. The custom format is specified in the multi-file spec
(per pivot decision D6); see `../` (the parent `spec/` directory)
once the restructure lands.

The pivot rationale (short version):

- **Spec-first oracle.** The Python reference reader must implement
  from the spec alone. FlatBuffers runtime breaks this; implementing
  FlatBuffers from spec is the same effort as implementing custom.
- **Byte-addressability.** FlatBuffers' vtable indirection breaks
  direct-offset access needed for HTTP range streaming and mmap.
- **No per-record overhead.** FlatBuffers tables carry ~8 B per
  access of vtable overhead. For metadata with millions of inodes,
  that is MBs of overhead.
- **No external dependencies.** Custom format means no Google
  (FlatBuffers), no Facebook (Frozen2/folly), no Cap'n Proto LLC.
- **Spec-first purity.** The spec becomes the wire format itself —
  no translation layer through `.fbs`.

## What replaces these files

The custom wire format specified in the multi-file spec:

- Layer 2 (`wire-format/`) — section-level descriptions
- Layer 3 (`bit-level/`) — byte and bit layouts for every fixed-width
  type

The Rust crate `limnifs-format` implements both layers via `serde`;
Python/Ruby/TS adapters implement from spec (spec-only path) or wrap
Rust via FFI/WASM.

## Historical context

These files were valid FlatBuffers (CI's `flatc --schema --binary
--no-warnings` compiled both in 48s during session 5). They are kept
here as historical reference for the design evolution. They are NOT
authoritative for the wire format; the multi-file spec is.

## Do not use

- Do not generate bindings from these `.fbs` files.
- Do not reference them in new code.
- Do not update them to track format changes.

If you need the wire format, read the multi-file spec under
`spec/{concepts,wire-format,bit-level}/`.
