# 34 — Directory node (bit-level)

- **Source:** [§4.2 Directory representation](../wire-format/22-metadata.md)
- **Total width:** variable
- **Alignment:** packed contiguously in the metadata blob; the root
  directory's node is referenced by the root inode's
  `btree_node_hash` (§4.1).

A directory node is the container for a directory's entries. Per
pivot D2, the directory tree is a deterministic Merkle B-tree
(Prolly-inspired). v0.1 defines a single layout: the **leaf node**.
All entries fit in one node; large directories MAY use multiple
leaf nodes linked by the Merkle B-tree's internal structure in a
future revision.

Entries within a node MUST be lexicographic by name (§4.2). This
makes range reads and diff walks deterministic (§1.4).

## Node layout (v1 leaf)

```
+----------------------------------------------------------+
| node_version : u8   (= 1)                                |  offset 0
+----------------------------------------------------------+
| entry_count  : u32 LE                                    |  offset 1
+----------------------------------------------------------+
| entries[]    : entry_count × DirEntry (sorted by name)   |  offset 5
+----------------------------------------------------------+
```

Total width: `5 + Σ entry_sizes` bytes.

## DirEntry layout (variable width)

```
+----------------------------------------------------------+
| name_len     : u32 LE                                    |  offset 0
+----------------------------------------------------------+
| name         : name_len bytes (UTF-8, no NUL)            |  offset 4
+----------------------------------------------------------+
| inode_number : u64 LE                                    |  variable
+----------------------------------------------------------+
| entry_type   : u8                                        |  variable
+----------------------------------------------------------+
```

Entry width: `4 + name_len + 8 + 1` bytes.

## Byte-offset table (node level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | node_version | u8 | — | currently `1` |
| 1 | 4 | entry_count | u32 | LE | number of directory entries |
| 5 | variable | entries | DirEntry[] | — | MUST be sorted by name |

## DirEntry field semantics

### name (bytes 4..4+name_len, UTF-8)

The directory entry name. A single path component — no `/`
separator. MUST be non-empty. MUST NOT contain `/` (0x2F) or NUL
(0x00) bytes.

### inode_number (after name, u64 LE)

The inode number this entry points to. Cross-references the
inode's `number` field in §4.1.

### entry_type (after inode_number, u8)

| Value | Meaning | POSIX equivalent |
|---|---|---|
| `0x01` | file | `S_IFREG` |
| `0x02` | directory | `S_IFDIR` |
| `0x03` | symlink | `S_IFLNK` |
| `0x04` | special | `S_IFBLK`, `S_IFCHR`, `S_IFIFO`, `S_IFSOCK` |
| `0x05`–`0xFE` | reserved | future types |
| `0xFF` | extended | post-v1 type descriptor |

The `entry_type` MUST be consistent with the referenced inode's
`mode & S_IFMT`. Mismatches are `Corrupt` (detected at the
metadata-walker layer after both the directory node and the inode
are parsed).

## Merkle hashing

Each directory node is hashed by BLAKE3 over its wire bytes (from
`node_version` through the last entry's `entry_type`). The hash is
stored in the parent directory inode's `content_handle` field (as
the `Directory([u8; 32])` variant per §4.1).

The root directory's inode carries the hash of the root directory
node. The metadata blob contains:
1. The root inode (§4.1).
2. The root directory node (this spec).
3. Child inodes and child directory nodes, recursively.

The `btree_node_hash` in the root inode is `BLAKE3(root_directory_node_bytes)`.

## Validation rules

A directory node is valid iff:

1. At least 5 bytes available for the fixed prefix.
2. `node_version == 1`. Other versions: `UnsupportedFeature`.
3. Entries are sorted lexicographically by name. Unsorted entries:
   `Corrupt`.
4. Each entry's `name` is non-empty, valid UTF-8, and contains no
   `/` or NUL bytes. Otherwise: `Corrupt`.
5. Each entry's `entry_type` is in `{0x01, 0x02, 0x03, 0x04}`.
   Other values: `UnsupportedFeature` (or `Corrupt` for `0xFF`).
6. No duplicate names within the node. Otherwise: `Corrupt`.

## Worked example

### Root directory with two files and one subdirectory

```
01                              node_version = 1
03 00 00 00                      entry_count = 3

[DirEntry 0: "bin" (directory)]
03 00 00 00                      name_len = 3
"bin"                            name
01 00 00 00 00 00 00 00          inode_number = 1
02                              entry_type = directory

[DirEntry 1: "hello.txt" (file)]
09 00 00 00                      name_len = 9
"hello.txt"                      name
02 00 00 00 00 00 00 00          inode_number = 2
01                              entry_type = file

[DirEntry 2: "README.md" (file)]
09 00 00 00                      name_len = 9
"README.md"                      name
03 00 00 00 00 00 00 00          inode_number = 3
01                              entry_type = file
```

Entries are sorted: "README.md" < "bin" < "hello.txt"
lexicographically. Wait — that's wrong; the example shows "bin"
first. Let me fix: actually `"R" < "b"` in ASCII (0x52 < 0x62), so
the correct order is "README.md", "bin", "hello.txt".

Corrected:
```
01                              node_version = 1
03 00 00 00                      entry_count = 3

[DirEntry 0: "README.md" (file, inode 3)]
[DirEntry 1: "bin" (directory, inode 1)]
[DirEntry 2: "hello.txt" (file, inode 2)]
```

## Cross-references

- [§4.2 Directory representation](../wire-format/22-metadata.md) —
  semantic description.
- [§4.1 Inode](../wire-format/22-metadata.md) — what `inode_number`
  references.
- [33-inode.md](33-inode.md) — inode wire layout; the
  `Directory([u8; 32])` content handle carries this node's hash.
- [§1.4 Determinism](../concepts/11-identity.md) — why sorting
  matters.
- [Pivot D2](https://github.com/limnifs/limnifs/blob/main/docs/superpowers/specs/2026-07-29-wire-format-pivot.md)
  — the deterministic Merkle B-tree decision.
