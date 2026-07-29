### 3. Drop store (Layer 1)

This section is normative. The drop store is the bulk-data layer of a
`.limni` image; everything else is metadata.

#### 3.1 Slab format

A slab is a contiguous byte object in the drop store. It is the unit
of locator addressing for Layer 1. Slab structure:

```
+---------------------------------+
| SlabHeader (fixed, §3.2)        |   ← magic, version, slab_id, length, EC, crypto hint
+---------------------------------+
| DropRecord[0]   (§3.3)          |
| ...                             |
+---------------------------------+
| DropRecord[n-1]                 |
+---------------------------------+
| SolidWindow[0]  (§3.4)          |   ← concatenated compressed bytes for drops 0..k
| ...                             |
+---------------------------------+
| SolidWindow[m-1]                |
+---------------------------------+
| ECShards (optional, §3.5)       |   ← k+m shards when Reed-Solomon is enabled
+---------------------------------+
```

Slabs MUST be between 4 MiB and 64 MiB inclusive, unless the manifest's
parameters section overrides (see §5.4). Slabs MUST NOT span locator
entries; one slab maps to one primary locator plus optional mirror
entries.

#### 3.2 SlabHeader

The slab header is a fixed-size prefix at offset 0:

| Field | Type | Notes |
|---|---|---|
| magic | 4 bytes | `LIM1` (0x4C 0x49 0x4D 0x31) |
| format_version | u16 LE | bump-incompatible layout changes |
| slab_id | SlabId (40 bytes) | §2.2 |
| total_length | u64 LE | bytes including header |
| ec_descriptor | u8 | 0 = no EC; 1..N = EC id (matches §16); 255 = extended descriptor (post-v1) |
| crypto_hint | u8 | 0 = plaintext; N = AEAD id (§10); the actual key lives in the manifest (§5.5) |

`magic` lets readers reject misidentified bytes before parsing the
rest. `format_version` is per-layer; see §17 versioning.
`ec_descriptor = 0` means no erasure coding; readers MUST NOT attempt
reconstruction. `ec_descriptor = 255` means the extended descriptor
follows (post-v1).

The header is followed by drop records, then solid windows, then
optional EC shards. Field order is fixed.

Byte-level layout: see [bit-level/30-slab-header.md](../bit-level/30-slab-header.md).
Total header width: 4 + 2 + 8 + 32 + 8 + 1 + 1 = 56 bytes (the
`slab_id` is encoded as ordinal u64 LE followed by 32-byte hash).

#### 3.3 Drop records

Each drop within a slab has a `DropRecord` describing where its bytes
live. A `DropRecord` is:

| Field | Type | Notes |
|---|---|---|
| drop_id | DropId (32 bytes) | §1.1 |
| plaintext_len | u32 LE | bytes after codec/AEAD inverse; readers size the destination buffer from this |
| representation | 3 bytes | §2.2 (codec, aead, ec); `aead = 0` and `ec = 0` allowed when the slab is plaintext |
| solid_window_index | u8 | index of the solid window containing this drop's bytes |
| offset_in_window | u32 LE | byte offset within the decompressed solid window |
| len_in_window | u32 LE | byte length within the decompressed solid window |

The triple `(solid_window_index, offset_in_window, len_in_window)` tells
the reader where to find this drop's plaintext inside the decompressed
solid window. `solid_window_index` is needed because a slab MAY have
multiple solid windows (one per class group, see §20.1 decision).

`DropRecord`s are packed contiguously in declaration order. The total
record count equals the count of drops whose representations point at
this slab; readers MAY validate this against the manifest's slab index.

#### 3.4 Solid windows

A solid window is the codec output for one or more consecutive drops.
Per §20.1: per-slab solid windows only (cross-slab class groups deferred
to `solid-blocks-v2`).

A solid window's boundaries are recorded by:
- The `DropRecord`s that reference it (their `solid_window_index` field).
- The next window's first drop record, or end-of-slab.

The codec used is `representation.codec`. When `codec = 0` (store), the
window's bytes are the literal concatenation of drop plaintexts, in
order, with no framing; readers MUST consume exactly `len_in_window`
bytes per drop.

When `codec != 0`, the window is compressed; readers decompress once
per window and then index by `(offset_in_window, len_in_window)`.

#### 3.5 EC shards (optional)

When `SlabHeader.ec_descriptor != 0` and `!= 255`, the slab's last `m`
blocks of size `ceil(|payload| / k)` are Reed-Solomon parity shards
(`k+m` total). See §16 and `02-algorithms.md §7` for the exact shard
field layout and reconstruction procedure.

The manifest's slab index (§5.4) MUST list the shard records for any
EC-enabled slab; readers resolve drops through the locator layer and
reconstruct only when fewer than `k` data shards are available.
