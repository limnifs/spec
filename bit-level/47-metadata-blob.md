# 47 — Metadata blob (bit-level)

- **Source:** [§4 Layer 2 metadata](../wire-format/22-metadata.md)
- **Total width:** variable
- **Alignment:** packed contiguously in the metadata reference
  section's `inline_metadata` payload (§5.3), or fetched via the
  metadata reference's locators.

The metadata blob is the layer-2 payload of a `LimniFS` image: every
inode and every directory node, packed contiguously. The metadata
reference section (§5.3) carries either the inlined blob bytes or
the locators that fetch them; either way, the resulting bytes are
this layout.

## Blob layout (v0.1)

```
+----------------------------------------------------------+
| inode_count    : u32 LE                                   |  offset 0
+----------------------------------------------------------+
| inodes[]       : inode_count × Inode (§4.1, 33-inode.md)  |  offset 4
+----------------------------------------------------------+
| dir_node_count : u32 LE                                   |  variable
+----------------------------------------------------------+
| dir_nodes[]    : dir_node_count × DirectoryNode           |  variable
|                  (§4.2, 34-directory-node.md)             |
+----------------------------------------------------------+
```

Total width: `4 + Σ inode_sizes + 4 + Σ dir_node_sizes` bytes.

The blob is intentionally simple: two length-prefixed lists, inodes
first, directory nodes second. The order within each list is the
writer's choice. Readers MUST NOT rely on a particular ordering;
they look up entries by `inode_number` or by `btree_node_hash`.

## Byte-offset table

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 4 | inode_count | u32 | LE | number of inode records |
| 4 | variable | inodes | Inode[] | — | packed contiguously (33-inode.md) |
| 4+Σ | 4 | dir_node_count | u32 | LE | number of directory node records |
| 8+Σ | variable | dir_nodes | DirectoryNode[] | — | packed contiguously (34-directory-node.md) |

## Field semantics

### inode_count (offset 0..4, u32 LE)

The number of [`Inode`](33-inode.md) records that follow. Every
inode referenced by every directory entry in `dir_nodes` MUST have
exactly one record in `inodes`; cross-section consistency runs at
the slab-walker layer after both halves of the blob are parsed.

Zero is valid for a degenerate blob; the writer does not currently
emit zero-inode blobs but readers MUST accept them.

### inodes (offset 4.., variable)

`inode_count` consecutive [`Inode`](33-inode.md) records. Each is
variable-width (depends on flags + content handle). The writer's
traversal order is reflected here — directory inodes are pushed
post-order (after their children) — but readers MUST NOT rely on
this.

### dir_node_count (after inodes, u32 LE)

The number of [`DirectoryNode`](34-directory-node.md) records that
follow. MUST equal the count of inodes whose `mode & S_IFMT ==
S_IFDIR`. Cross-section consistency is checked at the slab-walker
layer.

### dir_nodes (after count, variable)

`dir_node_count` consecutive [`DirectoryNode`](34-directory-node.md)
records. Each is variable-width (depends on entry count and name
lengths).

## Validation rules

A metadata blob is valid iff:

1. At least 8 bytes are available for the two count prefixes.
2. Each `inode` record parses per [33-inode.md](33-inode.md).
3. Each `dir_node` record parses per [34-directory-node.md](34-directory-node.md).
4. Every directory inode's `ContentHandle::Directory(hash)` references
   exactly one directory node whose wire-bytes hash to `hash`. The
   check runs after both halves of the blob are parsed.
5. Every directory entry's `inode_number` references an inode that
   appears in the `inodes` list. Otherwise: `Corrupt`.
6. Exactly one directory inode has no parent (i.e. its `number` is
   not referenced by any directory entry). That inode is the **root**.
   If zero or more than one candidate exists: `Corrupt` (multi-root
   images are not supported in v0.1).
7. The blob's BLAKE3 hash (computed over its full bytes) MUST equal
   the `metadata_hash` carried by the manifest's metadata reference
   section (§5.3). Otherwise: `Corrupt`.

## Root identification

Because the blob's lists are unsorted, the root directory inode is
identified by elimination:

> The root is the unique directory inode whose `number` does not
> appear as `inode_number` in any directory entry.

This is `O(n × m)` in the worst case (n inodes × m directory
entries). For typical LimniFS images (n < 10⁶, m < 10⁶) the linear
scan is acceptable; future revisions may add a root-inode pointer
to the manifest header if profiling shows this is hot.

## Worked example

### Minimal blob: one directory with one file

A blob with two inodes (root directory + one regular file) and one
directory node:

```
02 00 00 00                      inode_count = 2

[Inode 0: regular file "hello.txt"]
02 00 00 00 00 00 00 00          number = 2
00 40 00 00                      mode = 0o100644 (regular, rw-r--r--)
00 00 00 00                      uid = 0
00 00 00 00                      gid = 0
00 00 00 00 00 00 00 00          mtime = 0
00 00 00 00 00 00 00 00          ctime = 0
01 00 00 00                      nlink = 1
04                              flags = 0x04 (INLINE_DATA)
05 00 00 00                      inline_len = 5
"hello"                          inline data

[Inode 1: root directory]
01 00 00 00 00 00 00 00          number = 1
00 00 00 ED                      mode = 0o040755 (directory, rwxr-xr-x)
00 00 00 00                      uid = 0
00 00 00 00                      gid = 0
00 00 00 00 00 00 00 00          mtime = 0
00 00 00 00 00 00 00 00          ctime = 0
02 00 00 00                      nlink = 2
00                              flags = 0
C5 A1 E8 ... 32 bytes            btree_node_hash (BLAKE3 of the dir node)

01 00 00 00                      dir_node_count = 1

[DirectoryNode 0: root, one entry]
01                              node_version = 1
01 00 00 00                      entry_count = 1
09 00 00 00                      name_len = 9
"hello.txt"                      name
02 00 00 00 00 00 00 00          inode_number = 2
01                              entry_type = file
```

The root inode is `1` (a directory not referenced by any entry).
The file inode is `2` (referenced by the entry `hello.txt`).

## Cross-references

- [§4 Layer 2 metadata](../wire-format/22-metadata.md) — semantic
  description.
- [§5.3 Metadata reference](../wire-format/23-manifest.md) — section
  that carries this blob's bytes (inlined) or its locators.
- [33-inode.md](33-inode.md) — inode wire layout.
- [34-directory-node.md](34-directory-node.md) — directory node wire
  layout.
- [§1.4 Determinism](../concepts/11-identity.md) — why directory
  entries are sorted within a node (the blob itself is unsorted).
- [38-metadata-reference.md](38-metadata-reference.md) — the
  metadata_hash this blob is expected to hash to.
