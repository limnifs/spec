# SPEC.md — superseded

This file was the single-file LimniFS format specification through
v0.1-draft (1359 lines, last touched 2026-07-29).

**It is now superseded by the multi-file spec.** Read
[README.md](README.md) instead — it's the entry point to the onion-
layered file tree.

## Why the split

Per the
[wire format pivot decision D6](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md),
the spec is restructured into multiple files organized in onion layers
(orientation → concepts → wire-format → algorithms → conformance →
reference). Each file is focused on one topic and small enough to read
in one sitting.

This file is kept (per the project's never-delete rule) as a
historical reference. It is NOT authoritative — the multi-file spec is.

## Where the content went

The single-file content was split across these files (alphabetical):

- `01-glossary.md` — §2 Terminology
- `algorithms/40-addressing.md` — §6 Two-level addressing
- `algorithms/41-overlays.md` — §7 Overlay chains
- `algorithms/42-derivations.md` — §8 Derivation operations
- `appendices/A-references.md` — Appendix A
- `appendices/B-change-log.md` — Appendix B
- `concepts/11-identity.md` — §1 Foundational invariants
- `conformance/19-conformance.md` — §19 Conformance
- `crypto/15-crypto.md` — §15 Cryptography
- `crypto/16-ec.md` — §16 Erasure coding
- `decisions/20-resolved.md` — §20 Resolved in v0.1
- `decisions/21-deferred.md` — §21 Deferred
- `examples/22-worked-examples.md` — §22 Worked examples
- `registries/09-format.md` — §9 Registry format
- `registries/10-aead.md` — §10 AEAD registry
- `registries/11-codec.md` — §11 Codec registry
- `registries/12-locator.md` — §12 Locator scheme registry
- `registries/13-classifier.md` — §13 Classifier class registry
- `registries/14-feature-flags.md` — §14 Feature-flag registry
- `versioning/17-versioning.md` — §17 Versioning policy
- `versioning/18-unknown-flag.md` — §18 Unknown-flag policy
- `wire-format/21-drop-store.md` — §3 Drop store
- `wire-format/22-metadata.md` — §4 Filesystem metadata
- `wire-format/23-manifest.md` — §5 Manifest
