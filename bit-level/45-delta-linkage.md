# 45 — Delta linkage section (bit-level)

- **Source:** [§5.8 Delta linkage](../wire-format/23-manifest.md)
- **Total width:** variable
- **Alignment:** immediately follows the DMS policy section (§5.7)
  when both are present; otherwise follows the EC params section
  (§5.6); otherwise follows the slab index (§5.4). In a minimal
  image with no optional sections before it, follows the slab index
  directly.
- **Section version:** 1.

The delta linkage section identifies this image as a delta against a
parent image and records the tree operations that transform the
parent's filesystem tree into this image's tree. When this section is
absent, the image is standalone (not a delta).

## Section layout (v1)

```
+----------------------------------------------------------+
| section_version : u8   (= 1)                             |  offset 0
+----------------------------------------------------------+
| base_root       : 32 bytes (parent ManifestRoot)         |  offset 1
+----------------------------------------------------------+
| tree_op_count   : u32 LE                                 |  offset 33
+----------------------------------------------------------+
| tree_ops[]      : tree_op_count × TreeOp                 |  offset 37
+----------------------------------------------------------+
```

Total width: `37 + Σ tree_op_sizes` bytes.

## TreeOp layout (variable width)

Each tree operation describes one filesystem mutation against the
parent's tree:

```
+----------------------------------------------------------+
| op_type    : u8 (0x01 = Add, 0x02 = Remove, 0x03 = Replace)|  offset 0
+----------------------------------------------------------+
| path_len   : u32 LE                                       |  offset 1
+----------------------------------------------------------+
| path       : path_len bytes (UTF-8, slash-separated)     |  offset 5
+----------------------------------------------------------+
| [if op_type != Remove]:
| inode_number: u64 LE                                      |  variable
+----------------------------------------------------------+
```

`Remove` operations carry only the path. `Add` and `Replace`
operations carry the path plus the new inode number.

## Byte-offset table (section level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | section_version | u8 | — | currently `1` |
| 1 | 32 | base_root | bytes | — | parent image's `ManifestRoot` |
| 33 | 4 | tree_op_count | u32 | LE | number of tree operations |
| 37 | variable | tree_ops | TreeOp[] | — | packed contiguously |

## Field semantics

### section_version (offset 0, u8)

Currently `1`. Bumped on bump-incompatible changes.

### base_root (offset 1..33, 32 bytes)

The parent image's `ManifestRoot` per §1.2. This is the identity of
the image the delta was built against. Readers MUST verify that the
parent image is available and that its `ManifestRoot` matches this
value before applying the tree operations.

### tree_op_count (offset 33..37, u32 LE)

Number of `TreeOp` entries. Zero is valid (a delta with no tree
changes — only representation changes like deepening).

### tree_ops (offset 37.., variable)

`tree_op_count` consecutive `TreeOp` records in the order the delta
builder produced them. The order matters: operations are applied
sequentially to the parent's tree.

## TreeOp field semantics

### op_type (byte 0, u8)

| Value | Name | Meaning |
|---|---|---|
| `0x01` | Add | Insert a new inode at `path` |
| `0x02` | Remove | Delete the inode at `path` |
| `0x03` | Replace | Replace the inode at `path` with a new one |
| `0x04`–`0xFE` | reserved | Future ops; readers reject with `UnsupportedFeature` |
| `0xFF` | extended | Post-v1 op descriptor follows |

Per §20.2, v0.1 does NOT support a first-class `Rename` op. Renames
are compiled to `Remove(old_path)` + `Add(new_path)` by the delta
builder.

### path (bytes 5..5+path_len, UTF-8)

A slash-separated absolute path from the image's root directory.
Example: `usr/bin/python3` (no leading slash; relative to root).

The path uses `/` (0x2F) as the separator. Path components MUST NOT
contain `/` or NUL bytes. Empty path components (double slash) are
`Corrupt`.

### inode_number (after path, u64 LE, only for Add/Replace)

The inode number of the new inode being inserted (Add) or replacing
the existing one (Replace). The inode's full metadata lives in the
metadata layer (§4.1); this field is the cross-reference.

For `Remove`, this field is absent.

## Validation rules

A delta linkage section is valid iff:

1. At least 37 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. Each tree op's `op_type` is in `{0x01, 0x02, 0x03}`. Other
   values: `UnsupportedFeature` (or `Corrupt` for `0xFF`).
4. Each tree op's `path` is non-empty, valid UTF-8, and contains no
   NUL bytes. Otherwise: `Corrupt`.
5. Each tree op's `path` has no empty components (no `//`). Otherwise:
   `Corrupt`.
6. `Add` and `Replace` ops carry the 8-byte `inode_number` field.
7. `37 + Σ tree_op_sizes` does not overrun the manifest buffer.

Cross-section consistency (e.g., "every Add inode_number references
an inode in the metadata layer") runs at the slab-walker layer after
the metadata is also parsed.

## Worked example

### Delta with one Add and one Remove

A delta that adds `/usr/bin/newtool` and removes `/usr/bin/oldtool`:

```
01                              section_version = 1
DE AD BE EF ... 32 bytes         base_root (parent ManifestRoot)
02 00 00 00                      tree_op_count = 2

[TreeOp 0: Add]
01                              op_type = 0x01 (Add)
10 00 00 00                      path_len = 16
"usr/bin/newtool"                path (16 bytes)
2A 00 00 00 00 00 00 00          inode_number = 42

[TreeOp 1: Remove]
02                              op_type = 0x02 (Remove)
10 00 00 00                      path_len = 16
"usr/bin/oldtool"                path (16 bytes)
```

### Delta with zero tree ops (representation-only change)

A deepening delta where no tree paths change but representations do:

```
01                              section_version = 1
[32 bytes base_root]
00 00 00 00                      tree_op_count = 0
```

37 bytes total. The metadata layer is unchanged from the parent; only
slab representations differ.

## Cross-references

- [§5.8 Delta linkage](../wire-format/23-manifest.md) — semantic
  description.
- [§8.1 Delta](../algorithms/42-derivations.md) — how deltas are built.
- [§20.2 Resolved: rename semantics](../decisions/20-resolved.md) —
  why v0.1 compiles renames to Remove+Add.
- [§1.2 Image identity](../concepts/11-identity.md) — `ManifestRoot`
  used as the parent reference.
- [§4.1 Inode](../wire-format/22-metadata.md) — what the
  `inode_number` references.
- [§5.9 History](../wire-format/23-manifest.md) — a `delta` history
  entry cross-references this section's `base_root`.
- [38-metadata-reference.md](38-metadata-reference.md) — the section
  that precedes this one (or slab index if no metadata ref).
