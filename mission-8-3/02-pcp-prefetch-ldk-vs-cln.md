# PCP Prefetch Timing — LDK vs CLN

## Background

Per-commitment points (PCPs) are public keys derived for each commitment
state. They are used by the counterparty to construct commitment
transaction output keys. PCPs have no security significance on their
own — the signer must still validate and sign every commitment transaction
regardless of which PCPs have been released.

The VLS V2 proxy protocol batches multiple VLS calls into compound
messages to minimize round-trips over a slow transport link (e.g. nostr
to a secure enclave). The timing of PCP prefetches within these compound
messages differs between LDK and CLN due to their different internal
state machines.

## VLS PCP Policy

The signer enforces:

```
commitment_number <= next_holder_commit_num + 1
```

This allows fetching one PCP ahead of the current committed state.
At channel creation, `next_holder_commit_num = 0`, so PCP(0) and PCP(1)
are allowed. PCP(2) becomes available only after `ValidateCommitmentTx2(0)`
advances `next_holder_commit_num` to 1.

## CLN-Style Flow (Reference)

CLN requests PCPs when it actually needs them. This serves as a
reference point for understanding the LDK differences.

### Alice (Funder) — 4 RTTs

| Step | Compound message | PCPs fetched |
|------|-----------------|-------------|
| A01 | `vls_create_channel` | PCP(0) |
| A02 | `vls_funding_created` | — |
| A03 | `vls_confirm_counterparty_sig` | — (validate advances state) |
| A04 | `vls_channel_ready` | **PCP(1)** |

### Bob (Acceptor) — 3 RTTs

| Step | Compound message | PCPs fetched |
|------|-----------------|-------------|
| B01 | `vls_create_channel` | PCP(0) |
| B02 | `vls_funding_signed` | — (validate advances state) |
| B03 | `vls_channel_ready` | **PCP(1)** |

PCP(0) is fetched at channel creation. PCP(1) is fetched at channel
ready time, just before it is needed for the `channel_ready` wire
message (`second_per_commitment_point`).

## LDK-Style Flow (Design & Implementation)

LDK's `HolderCommitmentPoint` state machine maintains three points:

```
current:            PCP(N)     — current commitment
next:               PCP(N+1)   — sent to peer / ready for next state
pending_next_point: PCP(N+2)   — prefetch for pipeline
```

The critical constraint: `can_advance()` returns `true` only when
`pending_next_point` is `Some`. LDK will not send `channel_ready` or
`revoke_and_ack` unless `can_advance()` is true. This means LDK
requests PCPs **one step earlier** than CLN does.

### Alice (Funder) — 5 RTTs

| Step | Compound message | PCPs fetched |
|------|-----------------|-------------|
| A01 | `vls_create_channel` | PCP(0) + **PCP(1)** |
| A02 | `vls_funding_created` | — |
| A03 | `vls_confirm_counterparty_sig` | **PCP(2)** (validate advances state, then fetch) |
| — | `vls_sign_withdrawal` | — (funding TX signed by VLS via `VlsBdkSigner`) |
| A04 | `vls_channel_ready` | — (PCP(1) already cached from A01) |

### Bob (Acceptor) — 3 RTTs

| Step | Compound message | PCPs fetched |
|------|-----------------|-------------|
| B01 | `vls_create_channel` | PCP(0) + **PCP(1)** |
| B02 | `vls_funding_signed` | **PCP(2)** (validate advances state, SP appends fetch) |
| B03 | `vls_channel_ready` | — (PCP(1) already cached from B01) |

## Differences from CLN

### PCP(1) — fetched earlier

LDK calls `get_per_commitment_point(1)` immediately after
`derive_channel_signer`, before any wire messages are exchanged.
The implementation prefetches PCP(1) in `vls_create_channel` to
satisfy this. CLN waits until `channel_ready` time.

**Impact**: `vls_create_channel` grows from 3 calls to 4. The
`vls_channel_ready` message shrinks from 3 calls to 2 (no PCP needed).
Net effect: zero — same total calls, different distribution.

### PCP(2) — required before `channel_ready`

After receiving the counterparty's signature on commitment 0, LDK calls
`validate_holder_commitment`, then immediately calls
`get_per_commitment_point(2)`. LDK's `HolderCommitmentPoint` needs
PCP(2) as `pending_next_point` before `can_advance()` returns true and
`channel_ready` can be sent.

In the CLN-style flow, PCP(2) is not needed until the first commitment
update after the channel is open. There is no equivalent early request.

**Impact**: The `vls_confirm_counterparty_sig` compound validates the
counterparty signature and fetches PCP(2) in one round-trip. For Bob,
the PCP(2) fetch is appended to `vls_funding_signed` instead (the
validate is already included there).

### `vls_confirm_counterparty_sig` — Alice A03

When Alice receives `funding_signed(sig_Bf)`, the NP immediately sends
`vls_confirm_counterparty_sig` (ValidateCommitmentTx2 + GetPerCommitmentPoint(2)).
The validate advances the signer state, enabling the PCP(2) fetch in the
same round-trip. The NP caches ap2 and returns it on the subsequent
`get_per_commitment_point(2)` call from LDK.

`vls_sign_withdrawal` is a separate wallet-level compound message
sent by `VlsBdkSigner`. The BDK wallet is watch-only (xpub, no
xprv) — all signing goes through VLS.

Note: `vls_confirm_counterparty_sig` and `vls_sign_withdrawal` are
triggered by different LDK codepaths (commitment validation vs BDK
wallet signing), so they cannot be combined in the current ldk-node
architecture.

## Possible Optimization: Relax PCP Policy

If VLS relaxed its PCP policy to:

```
commitment_number <= next_holder_commit_num + 2
```

Then PCP(2) could be prefetched in `vls_create_channel` alongside PCP(0)
and PCP(1). This would:

- Eliminate `vls_confirm_counterparty_sig` as a separate step for Alice
  (validate could be folded into the SignWithdrawal compound)
- Reduce Alice from 5 RTTs to 4 RTTs (A01, A02, vls_sign_withdrawal, A04)
- Simplify the NP — no need to fetch PCP(2) after validate
- With `vls_sign_withdrawal` folded in, Alice would reach 3 RTTs

**Security impact**: None. PCPs are public keys that do not authorize
anything. The signer still validates and signs every commitment
transaction. The current `next_holder_commit_num + 1` policy is
conservative with no security benefit over `next_holder_commit_num + 2`.

**Trade-off**: The signer would allow the node to share PCP(2) with
the counterparty before any commitment has been validated. The
counterparty could construct commitment 2 keys, but cannot produce a
valid commitment transaction without the signer's signature.

## Summary

| | CLN (Reference) | LDK (Design & Implementation) |
|---|---|---|
| PCP(0) | A01/B01 | A01/B01 |
| PCP(1) | A04/B03 (channel_ready) | A01/B01 (create_channel) |
| PCP(2) | Not prefetched | A03/B02 (after validate) |
| A03 compound | `vls_confirm_counterparty_sig` + `vls_sign_withdrawal` | Same |
| Alice RTTs | 4 | 5 |
| Bob RTTs | 3 | 3 |
| Alice RTTs (relaxed policy) | 4 | **4** |
