### 19. Conformance

This section is normative.

#### 19.1 Conformance vectors

Conformance vectors in `02-conformance` exercise every boundary
condition stated in this spec. A reader claiming spec v0.1 conformance
MUST pass all v0.1 vectors.

Vector classes:

- **Identity**: round-trip of `DropId` for various inputs (empty,
  1 byte, exactly chunk-of-tree boundary, > 4 GiB streaming).
- **Merkle**: section substitution, omission, base_root transplant
  (per `02-algorithms.md §3`).
- **AEAD**: deterministic nonce for known inputs; transplant
  same-slab, cross-slab, cross-image; `Integrity` propagation.
- **Codec**: deterministic encode for known inputs; round-trip for
  every registered codec.
- **EC**: lose each single shard; lose all `C(k+m, m)` subsets for
  k=4, m=2; corrupt one shard (hash catches); k-1 shards ⇒ clean
  error.
- **Unknown-flag**: required unknown → `UnsupportedFeature`;
  optional unknown → image still opens.
- **Delta diff**: deterministic `tree_ops` for known input pairs.
- **Flatten post-condition**: `resolve(flatten(chain)) == resolve(chain)`.
- **Turnover post-condition**: standalone, byte-identical tree,
  cancel-safe.
- **Deepen identity invariant**: `DropId` set preserved across deepen.

#### 19.2 Reference reader as oracle

The Python reference reader (`limnifs/limnifs-py`) is the
spec-sufficiency oracle. It is written **from the spec only** — it
never reads the Rust implementation. If the Rust reader and the
Python reader agree on all vectors, the spec is unambiguous; if they
disagree, the spec has a gap.

The Python reader is required to be independently implementable. The
conformance harness runs both readers and compares outputs.
