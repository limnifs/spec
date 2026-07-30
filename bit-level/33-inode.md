# 33 — Inode record (bit-level)

- **Source:** [§4.1 Inode](../wire-format/22-metadata.md)
- **Total width:** variable (`FIXED_PREFIX + optional atime + xattrs + content_handle`)
- **Alignment:** packed contiguously in the metadata layer; the root
  directory's inode appears first.

An inode represents one filesystem object. Every entry in the
directory tree references an inode by its `number`. The inode
carries POSIX attributes (mode, owner, timestamps), optional
xattrs, and a type-dependent content handle.

## Fixed prefix (41 bytes)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 8 | number | u64 | LE | inode number, unique within the image |
| 8 | 4 | mode | u32 | LE | POSIX mode bits (file type + permissions) |
| 12 | 4 | uid | u32 | LE | owner user ID |
| 16 | 4 | gid | u32 | LE | owner group ID |
| 20 | 8 | mtime_ns | u64 | LE | modification time, nanoseconds since epoch |
| 28 | 8 | ctime_ns | u64 | LE | inode change time |
| 36 | 4 | nlink | u32 | LE | hard-link count |
| 40 | 1 | flags | u8 | — | bitfield controlling optional fields |

Total fixed prefix: **41 bytes**.

## Flags byte (offset 40, u8)

| Bit | Mask | Name | Meaning when set |
|---|---|---|---|
| 0 | `0x01` | `ATIME_PRESENT` | `atime_ns` field follows (8 bytes) |
| 1 | `0x02` | `HAS_XATTRS` | xattr block follows after optional atime |
| 2 | `0x04` | `INLINE_DATA` | For regular files: inline data is present in the content handle |
| 3–7 | `0xF8` | reserved | MUST be zero; readers reject nonzero with `Corrupt` |

When `ATIME_PRESENT` is clear, readers treat `atime` reads as
returning `mtime_ns` (§4.5).

## Optional atime (8 bytes, present iff flags bit 0)

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 41 | 8 | atime_ns | u64 | LE |

## Xattr block (variable, present iff flags bit 1)

When `HAS_XATTRS` is set, the xattr block follows the fixed prefix
(and optional atime):

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 4 | xattr_count | u32 | LE |
| 4 | variable | entries | XAttr[] | — |

Each `XAttr` entry:

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | namespace | u8 | — | 0x00=user, 0x01=trusted, 0x02=system, 0x03=security |
| 1 | 4 | key_len | u32 | LE | byte length of the key string |
| 5 | `key_len` | key | bytes | — | UTF-8, no NUL bytes |
| 5+`key_len` | 4 | value_len | u32 | LE | byte length of the value |
| 9+`key_len` | `value_len` | value | bytes | — | arbitrary |

## Content handle (variable, type-dependent)

After the fixed prefix + optional atime + optional xattrs, the
content handle follows. Its layout depends on the file type bits
(`mode & S_IFMT`):

### S_IFREG (regular file, 0x8000)

Two sub-cases controlled by `flags bit 2 (INLINE_DATA)`:

**Inline data** (`INLINE_DATA` set):

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 4 | inline_data_len | u32 | LE |
| 4 | `inline_data_len` | inline_data | bytes | — |

Default ceiling 4 KiB per inline blob (the threshold is spec-pinned
per §4.3; images MAY use a different threshold but the default is 4 KiB).

**Slice map** (`INLINE_DATA` clear):

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 4 | slice_count | u32 | LE |
| 4 | variable | slices | SliceRef[] | — |

Each `SliceRef`:

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 8 | file_byte_start | u64 | LE | start offset in the file |
| 8 | 8 | file_byte_end | u64 | LE | end offset in the file (exclusive) |
| 16 | 32 | drop_id | bytes | — | `DropId` (§1.1) |
| 48 | 4 | drop_byte_start | u32 | LE | offset within the drop's plaintext |
| 52 | 4 | drop_byte_len | u32 | LE | length within the drop's plaintext |

Each `SliceRef` is 56 bytes.

### S_IFDIR (directory, 0x4000)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 32 | btree_node_hash | bytes | — | BLAKE3 of the root Merkle B-tree node |

The directory's entries live in the Merkle B-tree (§4.2 / future
`bit-level/34-merkle-btree-node.md`). The `btree_node_hash`
references the root node's hash; readers resolve it via the
metadata layer's node index.

### S_IFLNK (symlink, 0xA000)

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 4 | target_len | u32 | LE |
| 4 | `target_len` | target | bytes | — |

UTF-8 target path string. MUST be a relative path per §4.3.

### S_IFBLK / S_IFCHR (block/char device, 0x6000/0x2000)

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 8 | dev | u64 | LE |

Device number (major/minor packed per POSIX convention).

### S_IFIFO / S_IFSOCK (fifo/socket, 0x1000/0xC000)

| Offset | Width | Field | Type | Endianness |
|---|---|---|---|---|
| 0 | 8 | pipe_id | u64 | LE |

Pipe or socket identifier.

## Validation rules

An inode is valid iff:

1. At least 41 bytes available for the fixed prefix.
2. `flags & 0xF8 == 0` (reserved bits clear). Otherwise: `Corrupt`.
3. `mode` encodes a valid POSIX file type (`S_IFREG`, `S_IFDIR`,
   `S_IFLNK`, `S_IFBLK`, `S_IFCHR`, `S_IFIFO`, `S_IFSOCK`). Unknown
   types: `Corrupt`.
4. If `ATIME_PRESENT` is set, at least 8 more bytes are available.
5. If `HAS_XATTRS` is set, the xattr block parses cleanly (each
   `key` is valid UTF-8 with no NUL; `namespace` is in
   `{0x00, 0x01, 0x02, 0x03}`).
6. The content handle matches the file type extracted from `mode`.
7. For `S_IFREG` with `INLINE_DATA`: `inline_data_len` does not
   exceed the configured ceiling (default 4 KiB).
8. For `S_IFREG` without `INLINE_DATA`: each `SliceRef` has
   `file_byte_start < file_byte_end`.

## Worked example

### Regular file with inline data (no xattrs, no atime)

A 10-byte file owned by root, mode 0644, no atime, no xattrs:

```
Fixed prefix (41 bytes):
  01 00 00 00 00 00 00 00       number = 1
  00 82 00 00                   mode = 0o100644 (S_IFREG | 0644)
  00 00 00 00                   uid = 0 (root)
  00 00 00 00                   gid = 0 (root)
  00 00 00 00 65 3E 84 00       mtime_ns = 1735000000 sec (example)
  00 00 00 00 65 3E 84 00       ctime_ns = same
  01 00 00 00                   nlink = 1
  04                            flags = 0x04 (INLINE_DATA, no atime, no xattrs)

Content handle (inline data):
  0A 00 00 00                   inline_data_len = 10
  [10 bytes of inline data]
```

Total: 41 + 4 + 10 = 55 bytes.

### Directory (no xattrs, no atime)

Root directory, mode 0755, nlink 2:

```
Fixed prefix (41 bytes):
  00 00 00 00 00 00 00 00       number = 0
  00 41 00 00                   mode = 0o040755 (S_IFDIR | 0755)
  00 00 00 00                   uid = 0
  00 00 00 00                   gid = 0
  00 00 00 00 65 3E 84 00       mtime_ns
  00 00 00 00 65 3E 84 00       ctime_ns
  02 00 00 00                   nlink = 2
  00                            flags = 0x00 (no atime, no xattrs, no inline)

Content handle (directory):
  [32 bytes btree_node_hash]
```

Total: 41 + 32 = 73 bytes.

## Cross-references

- [§4.1 Inode](../wire-format/22-metadata.md) — semantic description.
- [§4.2 Directory representation](../wire-format/22-metadata.md) —
  how directory inodes reference the Merkle B-tree.
- [§4.3 Content handles and slice maps](../wire-format/22-metadata.md)
  — slice map semantics.
- [§4.4 xattrs](../wire-format/22-metadata.md) — xattr namespace
  policy.
- [§4.5 atime omission](../wire-format/22-metadata.md) — the
  `ATIME_PRESENT` flag.
- [32-representation.md](32-representation.md) — `Representation`
  triple used in drop records.
- [31-drop-record.md](31-drop-record.md) — how `DropId`s referenced
  by slice maps are resolved.
