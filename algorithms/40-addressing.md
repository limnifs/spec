### 6. Two-level addressing

This section is normative. Two-level addressing is the load-bearing
resolution rule: it determines how a reader turns a byte-range
request on a logical file into a concrete byte sequence.

#### 6.1 Resolution chain

A read of byte range `[start, end)` of a slice proceeds in three steps:

1. **Slice → drops**: consult the slice map for the slice's containing
   inode (§4.3). Each entry is `(byte_range, slice_id)`. The range
   `[start, end)` is intersected with each slice's `byte_range`;
   matching slices are mapped to their `DropId`s via the manifest's
   slice map table.
2. **Drop → slab extent**: consult the slab index (§5.4). For each
   `DropId`, find the slab record that contains it; the extent is
   `(slab_id, offset_in_window, len_in_window, representation)` from
   the `DropRecord` (§3.3) plus the slab record's locator entries.
3. **Slab → bytes**: via the locator layer (§12), fetch the slab. The
   reader decompresses the relevant solid window(s) to obtain the drop
   plaintexts. The `DropSource` post-condition (§1.1) verifies
   `BLAKE3(decoded_plaintext) == DropId`.

The three steps MUST run in this order. Implementations MUST NOT skip
step 2 by guessing at slab offsets; the slab index is the only source
of truth for `DropId → slab extent`.

#### 6.2 Range read invariants

For a slice spanning drops `{d_0, d_1, …, d_n}` with each `d_i` having
`plaintext_len = L_i`, the slice's byte range `[start, end)` resolves
to:

- For each `d_i`, compute the intersection
  `[start, end) ∩ [prefix_sum_i, prefix_sum_{i+1})`.
- If non-empty, the partial drop `d_i[start_in_drop, end_in_drop]` is
  read.
- The reader MUST return bytes in slice order; partial drops MUST be
  reassembled into the requested range.

The reader MUST NOT inflate a full slab outside recorded solid blocks
(§20.1). Specifically:

- The reader MUST fetch only the slab extents overlapping the slice
  range; reading the entire slab is a bug.
- The reader MUST decompress only the solid windows overlapping the
  slice range; decompressing unrelated windows is a bug.

The `SlabRef` field order pinned in §2.2 (`SlabId`, `offset`, `len`,
`Representation`, then locator count + entries) is the only field
order readers accept. A `SlabRef` with fields in any other order is a
malformed `SlabRef`; readers reject with `Corrupt { offset, reason }`.
