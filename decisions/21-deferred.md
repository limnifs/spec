### 21. Deferred (owned by other components)

Two open design questions are deferred to other components per
CAMPAIGN.md. The spec carries the format invariants; the owning
component carries the parameters.

#### 21.1 FastCDC parameters and minimum drop size (design §16.2)

**Owner:** `04-writer-pipeline`. The manifest's parameter section
carries chunking parameters (min/avg/max sizes, gear table ID); the
format itself is parameter-agnostic — `DropRecord` (§3.3) records
`plaintext_len` per drop without assuming a chunking policy. `04`
chooses specific defaults with benchmarks; once chosen, this spec
records them as a referenced parameter set in §3.

#### 21.2 Time-lock puzzle calibration (design §16.3)

**Owner:** `05-crypto`. v0.1 ships Shamir k-of-n escrow only (§5.7
for the wire format). Time-lock puzzle code MUST NOT land until this
spec defines parameter selection and a Wesolowski/Pietrzak
proof-of-elapsed-time scheme. The `dms-time-lock` feature flag (§14)
is registered as optional and default-off; flipping it to standard
requires a spec amendment.
