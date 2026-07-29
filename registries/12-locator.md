### 12. Locator scheme registry

Initial contents:

| Scheme | Status | Reference | Notes |
|---|---|---|---|
| `file:` | standard (mandatory) | file path | local filesystem; no range streaming |
| `http:` | standard | RFC 7230 | range requests (§10.1 of design) |
| `https:` | standard | RFC 7230 | range requests; same wire format as `http:` over TLS |
| `s3:` | standard | AWS S3 API | multipart + conditional PUT for atomic write |
| `ipfs:` | standard | IPFS | CAR interop; drop names are multihash-compatible |
| `limni-p2p:` | experimental | design §10.1 | peer-to-peer locator |

Locator-entry wire format:

```
locator_entry = scheme ":" scheme_specific_part
```

`scheme_specific_part` is the scheme's documented URI/IRI form. For
`file:`, this is an absolute path. For `http(s):`, this is a URL with
optional range. For `s3:`, this is `bucket/key?region=...`. For `ipfs:`,
this is a CID. For `limni-p2p:`, this is a peer address plus content
hash.

`file:` is mandatory for v0.1 conformance. Locator racing (per
architecture §I9): when multiple entries exist for one slab, the
locator layer fetches from all in parallel; first successful bytes
win; lying locators are demoted via `Integrity` propagation.

Byte-level layout: see [bit-level/37-locator-entry.md](../bit-level/37-locator-entry.md).
Each locator entry is `[length: u32 LE][uri bytes]` where the URI
is the `scheme ":" scheme_specific_part` form documented above.
