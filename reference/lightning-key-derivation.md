# Lightning Key Derivation — From Diagram to BOLT #3

This document maps the simplified notation used in our diagrams to the actual key derivation formulas from the Lightning spec (BOLT #3).

---

## Notation Legend

Short notation used in all scenario diagrams. Alice's keys are uppercase `A` with a suffix, Bob's are uppercase `B`. Per-commitment values are lowercase with a state number.

### Static keys (exchanged once at channel open)

| Notation | Full name | Sent in |
|----------|-----------|---------|
| `Af` / `Bf` | `funding_pubkey` | open_channel / accept_channel |
| `Ar` / `Br` | `revocation_basepoint` | open_channel / accept_channel |
| `Ap` / `Bp` | `payment_basepoint` | open_channel / accept_channel |
| `Ad` / `Bd` | `delayed_payment_basepoint` | open_channel / accept_channel |
| `Ah` / `Bh` | `htlc_basepoint` | open_channel / accept_channel |

### Per-commitment values (change with each state N)

| Notation | Full name | How it's used |
|----------|-----------|---------------|
| `apN` / `bpN` | `per_commitment_point` (state N) | Public — shared with counterparty for key derivation |
| `asN` / `bsN` | `per_commitment_secret` (state N) | Private — revealed in `revoke_and_ack` when revoking state N |

### Derived keys (computed from basepoints + per-commitment point)

| Notation | Formula | Meaning |
|----------|---------|---------|
| `rp(Ar, bpN)` | revocation pubkey from Alice's basepoint + Bob's commitment point | Penalty path — Alice can spend if Bob broadcasts revoked state N |
| `dp(Bd, bpN)` | delayed payment key from Bob's basepoint + Bob's commitment point | Delayed path — Bob spends after CSV timeout |

### Other notation

| Notation | Meaning |
|----------|---------|
| `dt` | `to_self_delay` — CSV timelock in blocks |
| `H` | Payment hash (HTLC) |
| `P` | Payment preimage (satisfies H) |
| `T` | CLTV expiry (absolute block height) |
| `sig_Af(tx)` | Signature on `tx` using Alice's funding key |

---

## Basepoints Exchanged at Channel Open

Each side sends these static public keys in `open_channel` / `accept_channel`:

| Basepoint | Purpose |
|-----------|---------|
| `funding_pubkey` | 2-of-2 multisig for the funding output |
| `revocation_basepoint` | Derives revocation keys (penalty path) |
| `delayed_payment_basepoint` | Derives to_local delayed keys |
| `payment_basepoint` | Used directly for to_remote (static) |
| `htlc_basepoint` | Derives keys used in HTLC outputs |

These are generated from the node's seed via BIP-32 derivation. In VLS, this is what `GetChannelBasepoints` returns.

---

## Per-Commitment Secrets

Each commitment state has a `per_commitment_secret` (256-bit scalar), from which:

```
per_commitment_point = per_commitment_secret * G
```

The point is public (shared with the counterparty for key derivation). The secret is private and only revealed when revoking that state — this is the `revoke_and_ack` message.

In VLS, `GetPerCommitmentPoint` returns the point, and `RevokeCommitmentTx` reveals the secret.

### Shachain: Efficient Secret Storage

The per-commitment secrets are generated from a single seed using a hash-based tree (shachain). This allows:
- The generator to produce secrets in reverse order (newest first)
- The receiver to store all revealed secrets in O(log n) space
- Derivation of any previous secret from a later one

---

## Key Derivation Formulas

### Simple Derivation (delayed key, HTLC key)

Used for `to_local` delayed path and HTLC keys:

```
derived_key = basepoint + SHA256(per_commitment_point || basepoint) * G
```

The corresponding private key:

```
derived_privkey = basepoint_secret + SHA256(per_commitment_point || basepoint)
```

The holder can always compute this — they know their own `basepoint_secret` and the `per_commitment_point` (they generated it).

### Revocation Key (two-party derivation)

The revocation key is special — it requires material from **both** parties to spend:

```
revocation_pubkey = revocation_basepoint * SHA256(revocation_basepoint || per_commitment_point)
                  + per_commitment_point * SHA256(per_commitment_point || revocation_basepoint)
```

The corresponding private key:

```
revocation_privkey = revocation_basepoint_secret * SHA256(revocation_basepoint || per_commitment_point)
                   + per_commitment_secret * SHA256(per_commitment_point || revocation_basepoint)
```

This is why revocation works:
- **Before revocation:** The counterparty has `revocation_basepoint_secret` but not `per_commitment_secret` — cannot sign
- **After revocation:** The holder reveals `per_commitment_secret` in `revoke_and_ack` — now the counterparty has both pieces and can sign the penalty transaction

### Static Remote Key (no derivation)

With the `static_remotekey` feature (standard since 2019):

```
to_remote_key = payment_basepoint
```

No derivation at all. This enables data-loss recovery — the non-broadcaster only needs their seed to sweep funds.

---

## Mapping Diagram Notation to BOLT #3

Using Bob's commitment tx as an example (Bob holds this, could broadcast):

### Diagram notation

```
to_local:   A + b1 | B + dt          (simplified)
to_local:   rp(Ar, bs1) | dp(Bd, bp1) + dt   (explicit)
to_remote:  A                         (static)
```

### Actual BOLT #3 derivation

**to_local — revocation path** `rp(Ar, bs1)`:

```
revocation_pubkey = Ar * SHA256(Ar || bp1) + bp1 * SHA256(bp1 || Ar)

where:
  Ar  = Alice's revocation_basepoint (static, from open_channel)
  bp1 = Bob's per_commitment_point for state 1 (public)
  bs1 = Bob's per_commitment_secret for state 1 (revealed on revocation)
```

To spend (penalty): Alice computes `revocation_privkey` using her `revocation_basepoint_secret` + Bob's revealed `bs1`.

**to_local — delayed path** `dp(Bd, bp1) + dt`:

```
local_delayed_pubkey = Bd + SHA256(bp1 || Bd) * G

where:
  Bd  = Bob's delayed_payment_basepoint (static, from accept_channel)
  bp1 = Bob's per_commitment_point for state 1
```

To spend (normal): Bob waits `dt` blocks (CSV), then signs with his derived private key.

**to_remote** `A`:

```
to_remote_key = Ap

where:
  Ap = Alice's payment_basepoint (static, from open_channel)
```

To spend: Alice signs with her `payment_basepoint_secret`. No derivation needed.

---

## Summary Table

| Output | Key | Changes per state? | Why? |
|--------|-----|--------------------|------|
| to_local (revocation) | `rp(Ar, bpN)` | Yes | Encodes revocation secret — must be unique per state |
| to_local (delayed) | `dp(Bd, bpN)` | Yes | Original design choice, minor privacy benefit |
| to_remote | `Ap` | No | Made static for data-loss recovery |
| HTLC keys | `derived(htlc_basepoint, bpN)` | Yes | Same derivation as delayed key |
| Funding | `A_funding + B_funding` | No | Static 2-of-2 multisig |

Note: The delayed key changing per state is a design vestige. It could be static without affecting security — the revocation mechanism doesn't depend on it. It was kept as-is because only `to_remote` caused real recovery problems in practice.
