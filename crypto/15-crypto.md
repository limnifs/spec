### 15. Cryptography

This section is normative for wire-format-relevant invariants. Full
algorithm specs live in component `05-crypto`.

#### 15.1 Image key

The image key is a 32-byte random value, generated per image. It is
the secret from which all per-drop nonces derive. The image key is
wrapped per recipient via HPKE (X25519-DHKEM, HKDF-SHA256, ChaChaPoly
per RFC 9180).

| Field | Type | Notes |
|---|---|---|
| image_key | 32 bytes | per-image secret |
| recipients | repeated HPKEEnvelope | one per authorized reader |

Adding a recipient appends an envelope (image key unchanged, drops
untouched). Removing a recipient drops an envelope. Residual access is
a stated threat-model property — re-key = new image version.

#### 15.2 AEAD application

Per `02-algorithms.md §5`:

```
nonce  = HKDF-BLAKE3(image_key, info = slab_id ‖ u64le(drop_index))[0..24]
ad     = manifest_root ‖ slab_id ‖ u64le(drop_index)
ct     = aead.seal(image_key, nonce, ad, plaintext)
verify = aead.open(...); BLAKE3(decoded_plaintext) == drop_id
```

Deterministic nonces are safe because data is immutable: `(key, slab,
index)` tuples are unique by construction. Mutable data would forbid
this construction.

Transplant resistance: moving ct to another slab/index/image changes
AD ⇒ open fails. Conformance vectors cover transplant same-slab,
cross-slab, cross-image.

#### 15.3 Signatures (optional)

A manifest MAY carry a sigstore-compatible signature bundle over the
`ManifestRoot`. Signature is optional; integrity is always present via
BLAKE3 verification.

When a signature bundle is present:

| Field | Type | Notes |
|---|---|---|
| signature | bytes | sigstore signature bytes |
| certificate | bytes | signing certificate (Fulcio-issued) |
| transparency_log | bytes | Rekor inclusion proof (optional but recommended) |
| policy | string | OPA-style policy reference (e.g., `slsa-provenance-v0.2`) |

A reader with the verification key validates the signature against
the `ManifestRoot`. A reader without the key treats the signature as
informational (presence is recorded but not enforced).
