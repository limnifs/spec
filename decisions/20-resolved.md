### 20. Resolved in v0.1

The two open design questions owned by `01-spec` (design §16.1 and
§16.4) are resolved here. Both resolutions choose the *reversible*
option: the richer form can ship behind a feature flag in a future
spec version without breaking v0.1 readers.

#### 20.1 Solid-block boundaries (design §16.1)

**Decision:** per-slab solid windows with explicit boundaries in
metadata (see §3.4 for the wire format).

A solid block is a codec output that concatenates consecutive drops
within a single slab into one compressed stream. The `DropRecord`
(§3.3) records each drop's `(solid_window_index, offset_in_window,
len_in_window)` — the window index identifies which window in the slab
contains the drop's bytes; the offset and length pin the drop's slice
within the decompressed window. A slab MAY contain multiple solid
windows (one per class group); each window's boundaries are recorded
by the `DropRecord`s that reference it plus the next window's first
drop record.

**Rationale:** keeps slab addressing clean — every drop's bytes
resolve to `(slab_id, offset, len)` within one slab. Cross-slab class
groups (the alternative) would require a new addressing indirection
and a multi-slab inflation path; they are deferred to the
`solid-blocks-v2` feature flag (§14). Adding that flag later does not
break v0.1 readers (they treat it as required-unknown and reject, or
optional-unknown and fall back to per-slab windows per §18).

**Cost:** small-files ratio within a single slab. Cross-slab class
locality is the deeper optimization deferred to v2.

#### 20.2 Rename semantics in delta manifests (design §16.4)

**Decision:** v0.1 has NO first-class `Rename` op. The delta builder
MUST compile detected renames to `Remove(from)` + `Add(to)` pairs
(see §5.8 for the wire format).

**Rationale:** the v0.1 delta op set is `Add / Remove / Replace` only
(§5.8). Rename detection still happens at delta build time (the writer
walks add/remove pairs and matches identical content id sets); only
the encoding choice changes. The decision is reversible: a future
`rename-ops` feature flag (§14) can add `Rename(from, to, content_id)`
without breaking v0.1 readers (the flag is optional-unknown to them;
they fall back to compiled Remove+Add form per §18).

**Cost:** delta manifests for rename-heavy workloads are slightly
larger than first-class rename would be (two ops + duplicated content
id vs. one op). Recoverable in v2 if Phase 1+ benchmarks demand.
