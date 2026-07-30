# LimniFS — format specification

The single source of truth for the LimniFS `.lim` wire format. Everything
that two implementations must agree on lives here and nowhere else.

- **Spec version:** 0.1 (semantic: first verifiable reader target —
  Phase 0 exit gate is two independent readers passing the v0.1 vectors)
- **Status:** draft, self-sufficient as of 2026-07-29 (Parts I–VII
  cover identity, three layers, addressing, overlays, derivations,
  registries, crypto/EC, versioning, conformance, and resolved/deferred
  decisions)
- **File extension:** `.lim`
- **Wire format:** custom binary, owned by LimniFS (no FlatBuffers, no
  Avro, no Cap'n Proto — see
  [limnifs/limnifs:2026-07-29-wire-format-pivot.md](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md))

## How to read this spec

The spec is **onion-layered**: read at the depth you need.

- **Layer 0 — orientation** (start here): this `README.md`, then
  [01-glossary.md](01-glossary.md).
- **Layer 1 — concepts**: identity, layers, derivations. Read this to
  understand the system's design. See `concepts/`.
- **Layer 2 — wire format**: section-level descriptions of drop store,
  metadata, manifest, locators. Read this to navigate a `.lim` file at
  the section level. See `wire-format/`.
- **Layer 3 — bit-level**: byte-offset tables and bit-position diagrams
  for every fixed-width type. Read this to implement a parser. See
  `bit-level/`.
- **Layer 4 — algorithms**: read path, build path, deepen, delta,
  flatten, turnover, verify. Read this to implement a reader/writer.
  See `algorithms/`.
- **Layer 5 — conformance**: vectors, test format, reference reader
  contract. Read this to run conformance. See `conformance/`.
- **Layer 6 — reference**: registries (data), crypto/EC references,
  versioning, decisions, appendices. Lookup material.

A reader who only reads Layer 0 understands what LimniFS is and how to
use it. A reader who reads Layers 0–2 can navigate a `.lim` file. A
reader who reads Layer 3 can implement a parser. A reader who reads
Layer 4 can implement a full reader/writer. A reader who reads Layer 5
can run conformance.

## File map

### Layer 0 — orientation
- [README.md](README.md) — this file. Start here.
- [01-glossary.md](01-glossary.md) — terminology + semantic types.

### Layer 1 — concepts
- [concepts/11-identity.md](concepts/11-identity.md) — `DropId = BLAKE3(plaintext)`,
  image identity, representation plane, determinism.

### Layer 2 — wire format (the three layers)
- [wire-format/21-drop-store.md](wire-format/21-drop-store.md) — slabs, drop records,
  solid windows, Reed-Solomon shards.
- [wire-format/22-metadata.md](wire-format/22-metadata.md) — inodes, directory tree
  (deterministic Merkle B-tree), xattrs, slice maps.
- [wire-format/23-manifest.md](wire-format/23-manifest.md) — magic + header,
  nine sections, Merkle root construction.

### Layer 3 — bit-level
- [bit-level/README.md](bit-level/README.md) — how to read Layer 3; file index.
- [bit-level/30-slab-header.md](bit-level/30-slab-header.md) — the 56-byte slab
  header (§3.2): magic LIM1, format_version, slab_id (ordinal + hash), total_length,
  ec_descriptor, crypto_hint.
- [bit-level/31-drop-record.md](bit-level/31-drop-record.md) — the 48-byte drop
  record (§3.3): drop_id, plaintext_len, representation, solid_window_index,
  offset_in_window, len_in_window.
- [bit-level/35-manifest-header.md](bit-level/35-manifest-header.md) — the 16-byte
  manifest header (§5.1): magic, three u16 LE versions, six reserved bytes.
- [bit-level/36-feature-flags.md](bit-level/36-feature-flags.md) — the feature
  flags section (§5.2): section version, entry count, per-entry flag id + required.
- [bit-level/37-locator-entry.md](bit-level/37-locator-entry.md) — the length-prefixed
  locator URI (§12): u32 LE length + UTF-8 URI bytes.
- [bit-level/38-metadata-reference.md](bit-level/38-metadata-reference.md) — the
  metadata reference section (§5.3): BLAKE3 hash, locator entries, optional inline blob.
- [bit-level/39-slab-index.md](bit-level/39-slab-index.md) — the slab index section
  (§5.4): per-slab SlabId + locator lists.
- [bit-level/40-history.md](bit-level/40-history.md) — the history section (§5.9):
  per-entry op, timestamp, inputs, params.
- [bit-level/46-merkle-root.md](bit-level/46-merkle-root.md) — the Merkle root
  construction (§5.10): BLAKE3 over domain separator + 10 section hashes.

### Layer 4 — algorithms
- [algorithms/40-addressing.md](algorithms/40-addressing.md) — slice → drop →
  slab extent resolution; range-read invariants.
- [algorithms/41-overlays.md](algorithms/41-overlays.md) — overlay chains, depth
  limits, cycle detection, meromictic state.
- [algorithms/42-derivations.md](algorithms/42-derivations.md) — Delta, Flatten,
  Turnover, Deepen.

### Layer 5 — conformance
- [conformance/19-conformance.md](conformance/19-conformance.md) — vector classes,
  test format, Python reference reader contract.

### Layer 6 — reference

**Registries (data, OCP backbone):**
- [registries/09-format.md](registries/09-format.md) — registry row schema.
- [registries/10-aead.md](registries/10-aead.md) — AEAD algorithms.
- [registries/11-codec.md](registries/11-codec.md) — codecs.
- [registries/12-locator.md](registries/12-locator.md) — locator schemes.
- [registries/13-classifier.md](registries/13-classifier.md) — Seine classes.
- [registries/14-feature-flags.md](registries/14-feature-flags.md) — feature flags.

**Crypto and redundancy (references):**
- [crypto/15-crypto.md](crypto/15-crypto.md) — image key, AEAD application, signatures.
- [crypto/16-ec.md](crypto/16-ec.md) — Reed-Solomon layout, reconstruction trigger.

**Versioning:**
- [versioning/17-versioning.md](versioning/17-versioning.md) — per-layer version
  numbers, compatibility rules.
- [versioning/18-unknown-flag.md](versioning/18-unknown-flag.md) — required-unknown
  vs optional-unknown policy.

**Decisions (ADR-style):**
- [decisions/20-resolved.md](decisions/20-resolved.md) — solid blocks, rename
  semantics (resolved in v0.1).
- [decisions/21-deferred.md](decisions/21-deferred.md) — FastCDC parameters,
  time-lock calibration (deferred to other components).

**Examples and appendices:**
- [examples/22-worked-examples.md](examples/22-worked-examples.md) — stub
  walkthroughs (single image, delta chain, encrypted, EC).
- [appendices/A-references.md](appendices/A-references.md) — RFCs, papers, specs.
- [appendices/B-change-log.md](appendices/B-change-log.md) — spec version history.

## Deprecated

- [schema/DEPRECATED.md](schema/DEPRECATED.md) — FlatBuffers `.fbs` files are
  kept here as historical reference; the custom wire format specified in this
  multi-file spec supersedes them.

## What's authoritative

When this spec and any implementation diverge, **this spec is correct**.
When this spec and `00-architecture` (in `limnifs/limnifs`) diverge,
`00-architecture` wins for interface contracts; this spec wins for wire
bytes. When this spec and the design doc diverge, this spec wins.

## Source repositories

- **This spec** — [limnifs/spec](https://github.com/limnifs/spec).
- **Rust workspace** — [limnifs/limnifs](https://github.com/limnifs/limnifs).
  Includes `TODO.impl/` (work breakdown), `00-architecture/` (interface
  contracts), and `docs/superpowers/specs/` (design rationale + pivot
  decisions).
- **Python reference reader** — [limnifs/limnifs-py](https://github.com/limnifs/limnifs-py).
  Written from this spec alone; never reads the Rust code.
- **Legacy Frozen2 adapter** — [limnifs/limnifs-frozen2](https://github.com/limnifs/limnifs-frozen2).
  Reads existing DwarFS Frozen2 images; never writes.

## License

Apache-2.0 OR MIT. No GPL-3 anywhere.
