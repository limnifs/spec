### 1. Foundational invariants

This section is normative. A reader MUST implement every invariant stated
here. Implementation divergences invalidate spec conformance.

#### 1.1 The identity rule

`DropId` is the BLAKE3-derived identity of a drop's plaintext bytes:

```
DropId       = BLAKE3(plaintext)                      // 256-bit output
DropId.text  = "b3:" || base32_no_pad(DropId)        // multihash display
```

The `plaintext` argument is the drop's bytes AFTER full decoding through
every representation transform (codec inverse, AEAD open, EC
reconstruct). It MUST be the bytes the user would read from the
corresponding slice range if the image were mounted read-only. See
`02-algorithms.md §2` for the full decoding-then-verification sequence.

`DropId` is a 32-byte value (256 bits). Its display form is the multihash
text string `b3:<base32-lower-no-pad>` where `<base32-lower-no-pad>` is
RFC 4648 base32 lowercase without padding. The `b3:` prefix is the
multihash code for BLAKE3-256 (multihash table). This display form is
multihash-compatible and round-trips with `multibase` decoders, enabling
CAR import/export and IPFS placement without an additional adapter
layer (§12 IPFS locator).

A drop's `DropId` is independent of every representation choice. The same
plaintext sealed with XChaCha20-Poly1305 then zstd-compressed and the
same plaintext stored uncompressed under no AEAD share the SAME
`DropId`. Readers MUST verify `BLAKE3(decoded_plaintext) == DropId` after
all representation inversions; this hash check is the canonical test of
identity. The AEAD tag and the EC shard hash are fast-fail optimizations
that fire before the BLAKE3 check.

#### 1.2 Image identity

`ManifestRoot` is the identity of an entire `.limni` image. It is the
Merkle root over the layer-2 metadata blob and the manifest's own
sections, as defined in §5 Merkle root construction. Two images with the
same `ManifestRoot` are byte-for-byte equivalent under this spec: they
reference the same metadata, the same slabs, and the same drops with
the same representations.

`ManifestRoot` is the only handle by which an image is identified at rest
and in transit. URLs, container IDs, content hashes, and IPFS CIDs all
reference images by `ManifestRoot`. Distribution channels (HTTP, S3,
IPFS, P2P) MAY add their own identifiers for caching and deduplication,
but those identifiers MUST resolve to `ManifestRoot` and the resolved
image MUST byte-equal what `ManifestRoot` names.

#### 1.3 Representation plane separation

Codec, AEAD, erasure coding, and locator placement are collectively the
*representation plane*. They never appear in the identity plane.
Concrete consequences:

- A drop MAY have multiple `Representation` rows in the manifest's slab
  index, each pointing at a different `(slab_id, offset, len)` tuple
  with a different `(codec, aead, ec)` triple. All such rows decode to
  the same plaintext and therefore share `DropId` (§1.1).
- A reader's `DropSource` (architecture §I2) takes a `DropId` and a
  `Representation` as input. The reader selects one representation
  (locator racing per architecture §I9), decodes through it, and
  verifies `BLAKE3(decoded_plaintext) == DropId`. The hash verification
  is canonical; representation metadata is not.
- Deepening (representation upgrade from epilimnion to hypolimnion) is
  a strict representation-plane operation: it APPENDS a new
  `Representation` row and never mutates an existing one or changes any
  `DropId`.
- Locator mirroring is likewise representation-plane: a slab may have
  multiple locator entries pointing at the same `(codec, aead, ec)`
  triple stored at different physical locations.

#### 1.4 Determinism

Every byte of a `.limni` image MUST be a deterministic function of the
input tree and the manifest's recorded parameters, with two exceptions:
the image creation timestamp (in the history section, §5.9) and the
image key (in the crypto params section, §5.5). Concretely:

- **Chunking**: same input bytes + same FastCDC parameters
  (min/avg/max, gear table ID) ⇒ same set of drops. Writers MUST NOT
  introduce any nondeterministic source (random padding, system time)
  in the chunking pipeline.
- **Classification**: same drop head bytes + same classifier registry ⇒
  same class assignment.
- **Codec encode**: same input bytes + same codec id + same level ⇒
  same output bytes. The conformance harness in `02-conformance`
  verifies this.
- **Slab packing**: same drop sequence + same packing parameters ⇒
  same slabs in the same order. Slice traversal order during build is
  spec-fixed (lexicographic by slice key, §6.2).
- **Manifest assembly**: same inputs ⇒ same sections, same section
  hashes, same `ManifestRoot`.

A nondeterministic byte found in CI (same input, two runs, different
`ManifestRoot`) is a bug class tracked under `02-conformance`. The
conformance harness runs the build pipeline twice and asserts byte
equality of the resulting manifests.
