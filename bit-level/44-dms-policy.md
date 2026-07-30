# 44 — DMS policy section (bit-level)

- **Source:** [§5.7 DMS policy (optional)](../wire-format/23-manifest.md)
- **Total width:** variable
- **Alignment:** immediately follows the EC params section (§5.6) when
  both are present. Absent entirely when the image does not carry a
  Dead Man's Switch / key escrow record.
- **Section version:** 1.

The DMS policy section carries a Dead Man's Switch / key escrow
record. Per spec §5.7, v0.1 supports Shamir k-of-n secret sharing
only — time-lock puzzles are deferred (§21.2). The section is
OPTIONAL — it is present iff the DMS feature flag (`0x0002`) is
declared in the feature flags section (§5.2).

## Section layout (v1)

```
+----------------------------------------------------------+
| section_version         : u8   (= 1)                     |  offset 0
+----------------------------------------------------------+
| scheme                  : u8   (0x00 = Shamir)            |  offset 1
+----------------------------------------------------------+
| k                       : u8   (shares required)         |  offset 2
+----------------------------------------------------------+
| n                       : u8   (total shares)            |  offset 3
+----------------------------------------------------------+
| share_count             : u32 LE (MUST equal n)          |  offset 4
+----------------------------------------------------------+
| shares[]                : share_count × ShareRecord      |  offset 8
+----------------------------------------------------------+
| reconstruction_hint_len : u32 LE (0 if absent)           |  variable
+----------------------------------------------------------+
| reconstruction_hint     : reconstruction_hint_len bytes  |  variable
+----------------------------------------------------------+
```

The `share_count` field duplicates `n` (the total share count). Both
MUST be equal; readers reject mismatches as `Corrupt`. The
redundancy is intentional: future schemes may decouple them (e.g.,
a scheme where some shares are computed deterministically rather
than stored), so v1 carries both fields explicitly.

## ShareRecord layout (variable width)

```
+----------------------------------------------------------+
| custodian_id_len : u32 LE                                |  offset 0
+----------------------------------------------------------+
| custodian_id     : custodian_id_len bytes (UTF-8)        |  offset 4
+----------------------------------------------------------+
| share_data_len   : u32 LE                                |  variable
+----------------------------------------------------------+
| share_data       : share_data_len bytes                  |  variable
+----------------------------------------------------------+
```

`custodian_id` is a UTF-8 string identifying the custodian (e.g.,
`"alice@example.com"`, `"keycustodian-7"`). MUST be unique within
the section — each custodian receives exactly one share.
`share_data` is the Shamir share bytes; the length depends on the
secret's length (typically 32 bytes for a `ManifestRoot` or image
key).

## Byte-offset table (section level)

| Offset | Width | Field | Type | Endianness | Notes |
|---|---|---|---|---|---|
| 0 | 1 | section_version | u8 | — | currently `1` |
| 1 | 1 | scheme | u8 | — | only `0x00` (Shamir) supported in v0.1 |
| 2 | 1 | k | u8 | — | shares required to reconstruct; MUST be ≥ 1 |
| 3 | 1 | n | u8 | — | total shares; MUST satisfy `1 ≤ k ≤ n ≤ 255` |
| 4 | 4 | share_count | u32 | LE | MUST equal `n` |
| 8 | variable | shares | ShareRecord[] | — | packed contiguously |
| variable | 4 | reconstruction_hint_len | u32 | LE | 0 if no hint |
| variable | `reconstruction_hint_len` | reconstruction_hint | bytes | — | UTF-8, no NUL bytes |

## Field semantics

### section_version (offset 0, u8)

The layout version of this section. Currently `1`. Bumped on
bump-incompatible changes to the section or to `ShareRecord`.

### scheme (offset 1, u8)

The DMS scheme selector. v0.1 supports only Shamir k-of-n secret
sharing.

| Value | Meaning |
|---|---|
| `0x00` | Shamir k-of-n over GF(256) |
| `0x01`–`0xFE` | Reserved for future schemes (time-lock per §21.2, etc.) |
| `0xFF` | Extended scheme descriptor follows (post-v1) |

Readers reject any value other than `0x00` in v0.1 with
`UnsupportedFeature`.

### k (offset 2, u8)

The reconstruction threshold: how many shares are needed to
reconstruct the secret. `k ≥ 1`. `k = 1` means any single share
suffices (effectively no escrow); `k = n` means unanimous consent.

### n (offset 3, u8)

The total share count. `1 ≤ k ≤ n ≤ 255`. The upper bound is the
u8 width plus Shamir-over-GF(256)'s inherent limit (256 shares
maximum; v0.1 reserves one slot so 255 usable).

### share_count (offset 4..8, u32 LE)

The number of `ShareRecord` entries that follow. MUST equal `n`.
The redundancy with `n` is intentional; mismatches are `Corrupt`.

### shares (offset 8.., variable)

`share_count` consecutive `ShareRecord` records per the layout
above. Each `custodian_id` MUST be unique within the section; each
`share_data` is the raw Shamir share bytes.

### reconstruction_hint_len (variable offset, u32 LE)

Byte length of the optional reconstruction hint that follows. `0`
when no hint is present. Default ceiling 4 KiB; readers reject
larger values as `Corrupt`.

### reconstruction_hint (variable offset, `reconstruction_hint_len` bytes)

Optional UTF-8 human-readable note about reconstruction (e.g.,
`"contact legal@example.com for share assembly procedure"`). No
NUL bytes; readers MAY truncate for display.

## Validation rules

A DMS policy section is valid iff:

1. At least 8 bytes are available for the fixed prefix.
2. `section_version == 1`. Other versions: `UnsupportedFeature`.
3. `scheme == 0x00` (Shamir). Other values: `UnsupportedFeature`.
4. `1 ≤ k ≤ n ≤ 255`. Otherwise: `Corrupt`.
5. `share_count == n`. Otherwise: `Corrupt`.
6. Each share's `custodian_id` is non-empty and unique within the
   section. Empty or duplicate ids: `Corrupt`.
7. Each share's `share_data` is at least 1 byte and at most 1 KiB
   (Shamir share sizes are bounded by the secret length). Outside
   the range: `Corrupt`.
8. `reconstruction_hint_len` does not exceed the configured ceiling
   (default 4 KiB). Otherwise: `Corrupt`.
9. The reconstruction hint bytes are valid UTF-8 (when present).
   Otherwise: `Corrupt`.

## Worked example

### 3-of-5 Shamir escrow, no hint

A 3-of-5 Shamir scheme escrowing a 32-byte image key. The
reconstruction hint is absent.

```
01                              section_version = 1
00                              scheme = 0x00 (Shamir)
03                              k = 3
05                              n = 5
05 00 00 00                     share_count = 5 (matches n)

[ShareRecord 0]
06 00 00 00                     custodian_id_len = 6
"alice1"                        custodian_id
20 00 00 00                     share_data_len = 32
[32 bytes of share data]

[ShareRecord 1]
06 00 00 00                     custodian_id_len = 6
"bob123"                        custodian_id
20 00 00 00                     share_data_len = 32
[32 bytes of share data]

... 3 more ShareRecords ...

00 00 00 00                     reconstruction_hint_len = 0
```

### 2-of-3 with hint

A 2-of-3 scheme with a human-readable reconstruction hint:

```
01                              section_version = 1
00                              scheme = 0x00
02                              k = 2
03                              n = 3
03 00 00 00                     share_count = 3

[3 ShareRecords for "ceo", "cfo", "coo"]

36 00 00 00                     reconstruction_hint_len = 54
"Contact legal@example.com to coordinate share assembly."
```

## Cross-references

- [§5.7 DMS policy (optional)](../wire-format/23-manifest.md) —
  semantic description.
- [§15 Cryptography](../crypto/15-crypto.md) — where the image key
  comes from (the DMS section escrows it).
- [§21.2 Deferred: time-lock calibration](../decisions/21-deferred.md)
  — why v0.1 ships Shamir only.
- [§14 Feature-flag registry](../registries/14-feature-flags.md) —
  DMS flag `0x0002` that signals this section's presence.
