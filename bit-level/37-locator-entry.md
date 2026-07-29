# 37 — Locator entry (bit-level)

- **Source:** [§12 Locator scheme registry](../registries/12-locator.md)
- **Total width:** variable (`LENGTH_PREFIX_LEN + N` where N is the
  URI byte length; `LENGTH_PREFIX_LEN = 4`).
- **Alignment:** none (locators appear inside larger sections, e.g.
  the metadata reference §5.3 and the slab index §5.4).

A locator entry tells the reader where to fetch a blob. Each entry is
a length-prefixed URI in the form `scheme ":" scheme_specific_part`
per [§12](../registries/12-locator.md). Schemes are open: readers
MUST ignore schemes they do not implement when the locator is one of
several alternatives for the same blob (locator racing per §I9);
readers MUST fail with `UnsupportedFeature` when an unsupported
scheme is the sole locator for a required blob.

## Byte-offset table

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 4 | length | u32 | LE | byte length of the URI that follows |
| 4 | `length` | uri | bytes | — | UTF-8 URI: `scheme ":" scheme_specific_part` |

Total: `4 + length` bytes.

The length prefix is u32 LE (not a varint) for fixed-width simplicity
and to keep locator entries self-describing for locator racing — a
reader can scan past a locator whose scheme it does not implement in
O(1) by reading the length and skipping.

## Bit-position diagram

For a locator `file:///var/lib/limnifs/slab-7.bin` (31 bytes):

```
Byte  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15 ...
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     | 1F| 00| 00| 00| 'f' 'i' 'l' 'e' ':' '/' '/' '/' 'v' 'a' 'r' '/' ...
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     |<- length = 31 -->|<---------------- URI bytes (UTF-8) ------->
```

Each cell is one byte. The length 31 fits in one byte (0x1F); the
remaining three bytes of the u32 LE are zero.

## Field semantics

### length (bytes 0..4, u32 LE)

Byte length of the URI that follows. MUST be ≥ 4 (the smallest
meaningful URI is `x://` — a one-letter scheme plus `://`). Lengths
below 4 are `Corrupt`.

The maximum URI length is bounded by the locator-body-size ceiling
declared in the manifest's parameters section (default 4 KiB; the
spec may revise). Lengths above the ceiling are `Corrupt`.

### uri (bytes 4..4+length, UTF-8 bytes)

The locator URI. Per [§12](../registries/12-locator.md):

| Scheme | Reference | Example |
|---|---|---|
| `file:` | file path (mandatory baseline) | `file:///var/lib/limnifs/slab-7.bin` |
| `http:` | RFC 7230 | `http://cdn.example.com/slabs/7.bin` |
| `https:` | RFC 7230 over TLS | `https://cdn.example.com/slabs/7.bin` |
| `s3:` | AWS S3 API | `s3://my-bucket/slabs/7.bin?region=us-east-1` |
| `ipfs:` | IPFS (CID) | `ipfs://bafy...` |
| `limni-p2p:` | LimniFS peer protocol (experimental) | `limni-p2p://peer-id/content-hash` |

Readers parse the scheme by reading up to the first `:`. The
`scheme_specific_part` is everything after the colon; readers
interpret it per the scheme's reference.

## Validation rules

A locator entry is structurally valid iff:

1. At least 4 bytes are available for the length prefix. Shorter:
   `CoreError::TooShort`.
2. `length >= 4`. Otherwise: `Corrupt`.
3. `length` does not exceed the configured locator-body ceiling
   (default 4 KiB). Otherwise: `Corrupt`.
4. `4 + length` does not overrun the bytes available in the
   containing section. Otherwise: `CoreError::TooShort`.
5. The URI bytes contain at least one `:` byte (separating scheme
   from scheme-specific part). Otherwise: `Corrupt`.
6. The scheme (substring before the first `:`) is non-empty and
   contains only ASCII lowercase letters, digits, `+`, `-`, `.`
   (RFC 3986 `scheme = ALPHA *( ALPHA / DIGIT / "+" / "-" / "." )`).
   Otherwise: `Corrupt`.

Readers that do not implement the scheme SHOULD NOT reject the
entry on those grounds alone; they SHOULD record the entry and try
alternative locators (locator racing, §I9). Only when no
implementable locator remains for a required blob do they fail with
`UnsupportedFeature(scheme)`.

## Worked example

### Three locator entries for one slab

A slab mirrored to `file:`, `https:`, and `s3:` (locator racing
candidate). The three entries packed sequentially:

```
1F 00 00 00                                              <- length = 31
66 69 6C 65 3A 2F 2F 2F 76 61 72 2F 6C 69 62 2F ... 31 bytes total
"file:///var/lib/..."

28 00 00 00                                              <- length = 40
68 74 74 70 73 3A 2F 2F 63 64 6E 2E 65 78 61 6D ... 40 bytes total
"https://cdn.exam..."

21 00 00 00                                              <- length = 33
73 33 3A 2F 2F 6D 79 2D 62 75 63 6B 65 74 2F 73 6C ... 33 bytes total
"s3://my-bucket/s..."
```

Three entries × (4 + N) bytes = (4 + 31) + (4 + 40) + (4 + 33) =
120 bytes total.

### Malformed: missing colon

A 5-byte locator body `abcde` with no `:` separator:

```
05 00 00 00       <- length = 5
61 62 63 64 65    <- "abcde"
```

Reader rejects with `Corrupt { reason: "URI missing scheme separator" }`.

## Cross-references

- [§12 Locator scheme registry](../registries/12-locator.md) —
  semantic description and the scheme table.
- [§5.3 Metadata reference](../wire-format/23-manifest.md) — first
  section that uses locator entries.
- [§5.4 Slab index](../wire-format/23-manifest.md) — second section
  that uses locator entries (one set per slab).
- [§I9 (architecture)](https://github.com/limnifs/limnifs/blob/main/TODO.impl/00-architecture/00-overview.md)
  — locator racing: when multiple entries exist for one blob, the
  locator layer fetches from all in parallel; first successful
  bytes win; lying locators are demoted via `Integrity` propagation.
