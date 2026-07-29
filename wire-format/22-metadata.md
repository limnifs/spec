### 4. Filesystem metadata (Layer 2)

This section is normative for the semantics of every metadata field.
The binary encoding is FlatBuffers and is owned by
[01-flatbuffers-schema]; this section is the source of truth for what
each field MEANS.

#### 4.1 Inode

An inode represents a filesystem object. Every entry in the directory
tree references an inode. Inode fields:

| Field | Type | Notes |
|---|---|---|
| number | u64 LE | inode number, unique within the image |
| mode | u32 LE | POSIX mode bits (file type + permissions) |
| uid | u32 LE | owner user ID |
| gid | u32 LE | owner group ID |
| mtime_ns | u64 LE | modification time, nanoseconds since Unix epoch |
| ctime_ns | u64 LE | inode change time |
| atime_ns | u64 LE | access time (may be absent in some images; see §4.5) |
| nlink | u32 LE | hard-link count |
| xattrs | repeated XAttr | per-inode extended attributes (§4.4) |
| content_handle | ContentHandle | the actual data (§4.3) |

`number` is the inode's wire identifier and is stable across rebuilds
of the same logical filesystem (when content-derived; see §8 for how
inode numbers are assigned in delta chains).

`mode` MUST encode a valid POSIX file type (`S_IFREG`, `S_IFDIR`,
`S_IFLNK`, etc.). Readers MUST reject unknown file types with
`Unsupported(path)`.

`mtime_ns`, `ctime_ns`, `atime_ns` are nanoseconds since Unix epoch.
Readers MUST NOT assume second granularity; the value is exact.

#### 4.2 Directory representation

A directory inode's `content_handle` is a directory tree node.
[01-flatbuffers-schema] chooses between a B-tree layout and a flat
sorted array plus offset index. The spec leaves this open until the
schema task commits; whichever layout is chosen, the byte order of
entries within a directory MUST be lexicographic by name so that range
reads and diff walks are deterministic.

Directory entries: `(name: string, inode_number: u64 LE, entry_type: u8)`.
`entry_type` is `0x01 = file`, `0x02 = directory`, `0x03 = symlink`,
`0x04 = special` (block, char, fifo, socket).

#### 4.3 Content handles and slice maps

A regular file inode's `content_handle` is a slice map:

| Field | Type | Notes |
|---|---|---|
| inline_data | `Option<[u8]>` | present only for files at or below the inline-data threshold (§2.1) |
| slices | repeated `(byte_range, slice_id)` | ordered list of slices |

For files above the threshold, `inline_data` is absent and `slices`
contains the file's chunks in byte order. A `slice_id` resolves to a
`(byte_range_in_slice, DropId)` mapping in the manifest's slice map
table; readers follow the chain to fetch drops.

Symlink inodes: `content_handle` is a `redirect` field containing the
target path string (no slice map). Symlink targets MUST be relative
paths; absolute paths are resolved relative to the image's mount root.

Special inodes (block, char, fifo, socket): `content_handle` carries
the device number or pipe identifier; readers handle as appropriate.

#### 4.4 xattrs

Extended attributes are `(namespace, key, value)` triples:

| Field | Type | Notes |
|---|---|---|
| namespace | u8 | `0x00 = user`, `0x01 = trusted`, `0x02 = system`, `0x03 = security` |
| key | string | UTF-8, no NUL bytes |
| value | bytes | arbitrary |

Readers MUST enforce the namespace policy of the host OS (e.g.,
`security.SELinux` requires privilege).

#### 4.5 atime omission

Some images omit `atime` to save space (write-heavy workloads). When
omitted, readers MUST treat `atime` reads as returning `mtime_ns` and
MUST NOT report the omission as an error.

#### 4.6 Per-class records (Seine)

For each drop in the image, the metadata layer records the Seine
classification (§13) so deepening (§8.4) is reproducible:

| Field | Type | Notes |
|---|---|---|
| drop_id | DropId | §1.1 |
| class_id | u8 | Seine class registry §13 |

These records live alongside the slice map and are part of the
layer-2 metadata blob (so the Merkle root in §5.10 commits to them).
