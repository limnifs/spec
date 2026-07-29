### 8. Derivation operations

This section is normative. A derivation operation takes one or more
manifests and produces a new manifest. Each operation appends to the
new manifest's history (§5.9).

#### 8.1 Delta

A delta derives a child manifest from a base manifest:

```
delta(base, next) =
  Δtree_ops = diff(base.tree, next.tree)         // 02-algorithms §8
  Δdrops    = drops added by next (via 04-writer)
  manifest  = base manifest
              with delta_linkage { base_root: base.root, tree_ops: Δtree_ops }
              with slab_index extended with new drops' slab refs
              with history append { op: delta, inputs: [base.root] }
              with new Merkle root
```

The `tree_ops` are `Add | Remove | Replace` (no first-class `Rename` per
§20.2). The writer pipeline (04) computes the tree diff and produces
new drops for added/replaced content.

The diff algorithm MUST be deterministic — same `(base, next)` inputs
produce the same `tree_ops`. The conformance harness verifies this by
diffing twice and asserting equality.

#### 8.2 Flatten

Flatten derives a composite manifest from a chain, with zero drop-store
I/O:

```
flatten(chain [m_0 (base) … m_n]) =
  tree       = resolve(chain)                    // §7
  metadata   = tree.metadata
  slab_index = union of slab refs across chain
  manifest   = build_manifest(metadata, slab_index)
                with history append { op: flatten, inputs: chain.roots }
                with new Merkle root
```

Post-condition: `resolve(flatten(chain))` is byte-identical to
`resolve(chain)`. The conformance harness verifies this — a flatten
that produces a different resolved tree is a spec violation.

Performance: O(total metadata of chain). No data movement. Flatten is
the cheap operation; use it when defragmentation isn't needed.

#### 8.3 Turnover

Turnover derives a standalone manifest, repacking all drops into new
slabs:

```
turnover(chain, sink) =
  tree = resolve(chain)                          // §7
  for each slice in tree:
    for each drop_id in slice.slice_map:
      bytes    = read_drop(drop_id)              // from chain via 03
      new_rep  = chosen_rep(chain)               // per policy
      sink.put_drop(drop_id, new_rep, bytes)
  manifest = build_manifest(tree.metadata, sink.slabs)
              with history append { op: turnover, inputs: chain.roots }
              with new Merkle root
```

GC is implicit and exact: anything unreachable from the resolved tree
is never copied. Mark-and-sweep == copy-the-live-set. The conformance
harness verifies that every drop in the new manifest is reachable and
every reachable drop is in the new manifest.

Post-conditions:

- Standalone (no external slab references).
- Byte-identical tree to `resolve(chain)`.
- Cancel-safe: abandoning before `sink.commit` leaves no reader-visible
  state.

Performance: O(live data) I/O — the only tier that moves bytes.

#### 8.4 Deepen

Deepen re-encodes drops to better representations (e.g., LZ4 → zstd
for hypolimnion-class drops). Deepen is a strict representation-plane
operation:

```
deepen(image, policy) =
  for each drop in image where representation should change:
    bytes     = read_drop(drop_id)               // from image via 03
    new_rep   = policy.new_representation(drop, current_rep)
    sink.put_drop(drop_id, new_rep, bytes)       // append, not replace
  manifest = image manifest
              with slab_index extended with new representations
              with history append { op: deepen, inputs: [image.root] }
              with new Merkle root
```

Identity invariant: deepen APPENDS new representation rows. It MUST NOT
mutate or delete existing representation rows. The original `DropId`
relationships (§1.1) are preserved — the same plaintext still addresses
identically, with one or more representation rows pointing at it.

Performance: depends on policy. Per-drop cost is `O(plaintext_len)`
encode time for the new codec.
