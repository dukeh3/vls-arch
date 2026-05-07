# Lightning Protocol — How It Works

General reference for the Lightning protocol mechanics. Scenario docs assume familiarity with these concepts.

See also: [Key Derivation](lightning-key-derivation.md) for notation and key math, [Scenarios](scenarios.md) for the full roadmap.

---

## The Core Idea

A Lightning channel is a 2-of-2 multisig UTXO on the Bitcoin blockchain. Both parties pre-sign transactions that spend this UTXO, representing different balance distributions. Only the latest pre-signed transaction (the "commitment") is valid — older ones are revoked, meaning broadcasting them lets the counterparty take everything as a penalty.

This lets two parties update their balances off-chain, thousands of times, with only two on-chain transactions: the funding tx (open) and the close tx (cooperative or forced).

---

## Asymmetric, Independent Commitments

The two sides of a channel are **independent state machines**. There is no shared state. Each side has its own commitment transaction with its own commitment number, and the two sides do not need to be at the same number — ever.

Alice's commitment and Bob's commitment both spend the same funding output, but they differ in structure, commitment number, and which updates have been committed:

**Your own commitment (what you hold):**
- `to_local` — your balance, but delayed by a CSV timelock (`to_self_delay`). If you broadcast, you must wait.
- `to_remote` — counterparty's balance, immediately spendable by them.
- The delay exists so the counterparty has time to check if you broadcast a revoked state and punish you.

**Counterparty's commitment (what they hold):**
- Mirror image. Their balance is delayed, yours is immediate.

Both commitments also include a **revocation path** on the `to_local` output. If the holder broadcasts a revoked commitment, the counterparty can spend `to_local` immediately using the revocation secret — no waiting.

This asymmetry is fundamental: you always sign the *counterparty's* commitment (giving them their escape hatch), and they sign yours. The two commitments advance independently — Alice's commitment number can be ahead of or behind Bob's at any point in time.

---

## The Update-Commit-Revoke Cycle

Channel state advances through a three-phase cycle:

### Phase 1: Update — stage changes

Either side can send one or more `update_*` messages to propose changes. Both sides can send updates independently — Alice might send `update_add_htlc` while Bob sends `update_fulfill_htlc`. All pending updates from both sides are included when either side sends `commitment_signed`.

| Message | Purpose |
|---------|---------|
| `update_add_htlc` | Add an HTLC (conditional payment) |
| `update_fulfill_htlc` | Settle an HTLC by revealing the preimage |
| `update_fail_htlc` | Remove a failed HTLC |
| `update_fail_malformed_htlc` | Remove a malformed HTLC |
| `update_fee` | Change the commitment tx feerate (funder only) |

These messages **do not change the commitment** — they are staged in the local state of each peer, not on-chain and not signed. Nothing is committed or enforceable until `commitment_signed` is exchanged. Multiple updates can be batched before committing. For example, a routing node might receive three `update_add_htlc` messages before a single `commitment_signed` that commits all three at once.

### Phase 2: Commit — sign the new state

The sender packages all pending updates into a new commitment and signs it:

**`commitment_signed`** contains:
- A signature on the counterparty's new commitment tx
- One signature per HTLC output (the `htlc_sigs` — see below)

This gives the receiver a valid new commitment that includes all pending updates. The sender has not yet revoked their old state.

### Phase 3: Revoke — make the old state dangerous

The receiver responds with:

**`revoke_and_ack`** contains:
- `per_commitment_secret` — the secret for the OLD state (revoking it)
- `next_per_commitment_point` — the point for the state AFTER the new one

Once this is sent, the old state is revoked: if the sender of `revoke_and_ack` ever broadcasts the old commitment, the counterparty can compute the revocation key and take everything.

### The full round-trip

A typical state advance when Alice has updates to commit:

```
Alice                           Bob
  │                               │
  │── update_add_htlc ──────────►│  (1+ update messages)
  │── update_add_htlc ──────────►│
  │                               │
  │── commitment_signed ────────►│  Alice signs Bob's new commitment
  │                               │
  │◄── revoke_and_ack ──────────│  Bob revokes old state
  │                               │
  │◄── commitment_signed ───────│  Bob signs Alice's new commitment
  │                               │
  │── revoke_and_ack ───────────►│  Alice revokes old state
  │                               │
  Both old commitments are now revoked. Each side holds an updated commitment.
```

### Rules

- `commitment_signed` is always answered by `revoke_and_ack` from the other side. They are paired.
- You cannot send a second `commitment_signed` to the same peer until they have responded with `revoke_and_ack` for the previous one.
- Both sides can independently initiate `commitment_signed` — the protocol is not strictly turn-based. If both have pending updates, they can send `commitment_signed` concurrently. Each still gets its own `revoke_and_ack` in response.
- `update_*` messages have no immediate response. They accumulate until one side sends `commitment_signed`.

### The two sides are always independent

The two commitments are independent and don't need to match. At any point in time, Alice's commitment might include updates that Bob's doesn't, and vice versa. This is the normal state of affairs, not an edge case.

In the diagram above, after Bob sends `revoke_and_ack` but before Alice sends hers:
- Bob's commitment has advanced — it includes the pending updates, and his old commitment is revoked.
- Alice's commitment has not advanced yet — she still holds her previous commitment.

This is safe because each commitment is independently valid. Alice's old commitment is not revoked (she hasn't revealed her secret), and Bob's new commitment is properly signed. The protocol doesn't require — or even define — a moment where both sides are "in sync".

Each update (an HTLC add, a fulfill, a fee change) has its own state relative to each side's commitment. An HTLC can be committed on Bob's side but not yet on Alice's — this is a normal, expected part of the protocol, not a transient inconsistency. The BOLT spec calls these "pending" updates and tracks them per-side: an update is "committed on remote" as soon as the remote's `commitment_signed` includes it, and "committed on local" only when the local side's `commitment_signed` includes it. Both sides know which updates are committed where.

---

## HTLC Signatures (`htlc_sigs`)

Each HTLC on a commitment transaction creates an output that can only be spent via a **2nd-stage transaction** — either an HTLC-success tx (claim with preimage) or an HTLC-timeout tx (reclaim after expiry). These 2nd-stage transactions require signatures from *both* funding keys.

Since the commitment holder needs the counterparty's signature to spend their HTLC outputs, the counterparty pre-signs these 2nd-stage transactions and sends them alongside `commitment_signed`:

| HTLC type on holder's commitment | 2nd-stage tx | Pre-signed by counterparty |
|----------------------------------|-------------|---------------------------|
| Received HTLC | HTLC-success tx (claim with preimage) | Yes — holder adds their own sig + preimage to spend |
| Offered HTLC | HTLC-timeout tx (reclaim after CLTV expiry) | Yes — holder adds their own sig after timeout |

The `htlc_sigs` field in `commitment_signed` is an array — one signature per HTLC output, in the same order as the HTLCs appear on the commitment transaction.

---

## Channel Lifecycle

### Open

1. `open_channel` / `accept_channel` — exchange keys and parameters
2. `funding_created` / `funding_signed` — exchange signatures on initial commitments
3. Broadcast and confirm funding tx
4. `channel_ready` — exchange next per-commitment points, channel is live

### Operate

Repeat the update-commit-revoke cycle for each state change:
- Add HTLCs (route payments)
- Settle HTLCs (preimage flows back)
- Fail HTLCs (timeout or error)
- Adjust fees

### Close

**Cooperative** (both sides agree):
1. `shutdown` — both sides signal close, provide payout addresses, no new HTLCs
2. `closing_signed` — negotiate fee, exchange signatures on the close tx
3. Broadcast close tx — simple outputs, no timelocks

**Unilateral / Force close** (one side acts alone):
1. Broadcast your latest commitment tx
2. Wait for CSV delay on `to_local`
3. Sign and broadcast sweep tx to claim your funds
4. Claim any HTLC outputs via 2nd-stage txs

**Breach / Penalty** (counterparty cheats):
1. Counterparty broadcasts a revoked commitment
2. You detect it and have their revocation secret
3. Sign and broadcast penalty tx — takes everything from `to_local` using the revocation key

---

## What the Signer Sees

A VLS signer doesn't see the Lightning messages directly. The node translates each protocol event into signer API calls. The signer's job is to:

1. **Generate keys** — provide basepoints and per-commitment points
2. **Sign remote commitments** — produce signatures on the counterparty's commitment tx and HTLC 2nd-stage txs
3. **Validate local commitments** — verify the counterparty's signature on our commitment is correct
4. **Manage revocation** — release old secrets, validate received secrets
5. **Enforce policy** — reject anything that looks wrong (bad fees, unknown outputs, unbalanced HTLCs, revoked states)

See `current/01-alice-pays-bob.md` for how these calls map to each protocol step.
