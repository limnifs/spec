### 7. Overlay chains

This section is normative. An overlay chain is a sequence of manifests
where each non-base manifest declares `base_root` (§5.8) pointing to
its parent.

#### 7.1 Chain construction and resolution

A chain is built by applying delta operations (§8.1) to a base manifest.
Resolution walks the chain from the latest manifest back to the base,
merging tree operations in reverse:

1. Start at the leaf manifest (the one being opened).
2. For each manifest in the chain (leaf → base), apply its `tree_ops`
   to the resolved tree view.
3. Stop at the base (the manifest without `delta_linkage`).

Tree operations (`Add | Remove | Replace`, §5.8) are applied in the
order they appear in each manifest's `tree_ops` list. The resolved tree
is the union of the base tree and all applied operations, with later
operations in the chain overriding earlier ones.

The reader MUST apply tree operations deterministically: same chain
order, same operation order, ⇒ same resolved tree. The conformance
harness verifies this.

#### 7.2 Depth limits

The format imposes no depth limit on overlay chains. Depth is bounded
by reader policy; a default of `overlay_max_depth = 64` is recommended
(the value is set per-reader, not in the spec). When a reader's policy
is exceeded, it MUST return `Policy { rule: "overlay_max_depth" }`.

`overlay_max_depth` is not part of the wire format; it is a reader
configuration. Authors MAY choose chains deeper than any reader's
limit, but those chains are unreadable by that reader. Conformance
vectors include chain depths 1, 2, 5, and 64.

#### 7.3 Cycle detection

Manifests MUST NOT contain cycles in their `base_root` chain. A cycle
is when manifest A's chain (via `base_root`) returns to A.

Readers detect cycles by maintaining a set of seen `ManifestRoot`s while
walking the chain. If a `ManifestRoot` is encountered twice in a single
walk, the reader MUST reject with `Policy { rule: "overlay-cycle" }`.

#### 7.4 Meromictic state

A meromictic chain (§2.1) is one that is never flattened or turned
over. Meromictic chains are valid long-term state — readers MUST NOT
require flattening. Flatten (§8.2) is an optimization for performance,
not a correctness requirement.

A reader's read path through a meromictic chain has per-read cost
O(chain_depth) metadata, not O(data movement). This is the performance
profile that makes meromictic chains practical for long-tail use cases
(e.g., cumulative security patches).
