## Part VIII — Worked examples (stubs)

Each example walks through the bytes of a small image at v0.1. The full
byte-level walkthrough is added when the corresponding sections above
have complete prose AND the matching conformance vector exists. Outline
summaries only:

### 22.1 Single uncompressed image

One regular file, single drop, store codec, no crypto, no EC. Exercises:
manifest assembly, Merkle root, slab index with one entry, inode with
one slice-map entry.

### 22.2 Delta chain of depth 2

Base image (22.1) plus a delta adding one file and replacing another.
Exercises: `base_root` linkage, tree ops (Add, Replace), history
append, chain resolution.

### 22.3 Encrypted image (single recipient)

Like 22.1 with AEAD id = 0x01 (XChaCha20-Poly1305), one HPKE envelope.
Exercises: crypto params section, image key wrap, deterministic nonce
derivation, AD construction.

### 22.4 Erasure-coded image (k=4, m=2)

Like 22.1 with RS over the single slab. Exercises: EC params section,
shard records, reconstruction-and-verify path.

---
