# Scenario 01 — Alice Pays Bob (VLS Signer — Future/Batched)

Same scenario as `../reference/01-alice-pays-bob.md`, but showing VLS signer calls for both Alice and Bob. This is the **future/optimized** version where calls are batched where possible.

For notation (`Af`, `Ar`, `ap0`, `rp(...)`, etc.) see [Key Derivation](../reference/lightning-key-derivation.md#notation-legend).

---

## Commitment 0

Alice funds a 1.0 BTC channel.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer
    participant Bitcoin

    Alice->>SignerA: NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0)
    SignerA-->>Alice: Af, Ar, Ap, Ad, Ah, ap0

    Alice->>Bob: open_channel(Af, Ar, Ap, Ad, Ah, ap0, 1.0 BTC, to_self_delay=10)

    Bob->>SignerB: NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0)
    SignerB-->>Bob: Bf, Br, Bp, Bd, Bh, bp0

    Bob->>Alice: accept_channel(Bf, Br, Bp, Bd, Bh, bp0, to_self_delay=10)

    Alice->>SignerA: SetupChannel(is_outbound=true) + SignRemoteCommitmentTx(commitment_B_0)
    SignerA-->>Alice: sig_Af(commitment_B_0)

    Alice->>Bob: funding_created(funding_txid, output_index, sig_Af(commitment_B_0))

    Bob->>SignerB: SetupChannel(is_outbound=false) + ValidateCommitmentTx(commitment_B_0, sig_Af) + SignRemoteCommitmentTx(commitment_A_0)
    SignerB-->>Bob: OK, sig_Bf(commitment_A_0)

    Bob->>Alice: funding_signed(sig_Bf(commitment_A_0))

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_0, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: SignWithdrawal(funding_tx)
    SignerA-->>Alice: signed funding tx

    Alice->>Bitcoin: broadcast funding tx

    Bitcoin-->>Alice: confirmed
    Bitcoin-->>Bob: confirmed

    Alice->>SignerA: CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1)
    SignerA-->>Alice: ap1

    Alice->>Bob: channel_ready(ap1)

    Bob->>SignerB: CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1)
    SignerB-->>Bob: bp1

    Bob->>Alice: channel_ready(bp1)

    Note over Alice, Bob: Commitment 0 established<br/>Alice: 1.0 BTC | Bob: 0.0 BTC
```

### Signer Calls Summary

**Alice (funder) — 5 batched calls:**

| Call | When | Purpose |
|------|------|---------|
| NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) | Before open_channel sent | Get keys, register channel |
| SetupChannel + SignRemoteCommitmentTx(commitment_B_0) | After accept_channel received | Register counterparty params, sign Bob's commitment |
| ValidateCommitmentTx(commitment_A_0, sig_Bf) | After funding_signed received | Validate Bob's sig on our commitment |
| SignWithdrawal(funding_tx) | Before broadcast | Sign the funding tx inputs |
| CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1) | After funding tx confirmed | Lock channel, get ap1 for channel_ready |

**Bob (non-funder) — 3 batched calls:**

| Call | When | Purpose |
|------|------|---------|
| NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) | After open_channel received | Get keys, register channel |
| SetupChannel + ValidateCommitmentTx(commitment_B_0, sig_Af) + SignRemoteCommitmentTx(commitment_A_0) | After funding_created received | Register params, validate Alice's sig, sign her commitment |
| CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1) | After funding tx confirmed | Lock channel, get bp1 for channel_ready |

Note: SetupChannel requires both sides' parameters (funding_pubkey, basepoints, to_self_delay). It cannot be called until after accept_channel/funding_created — when both sides' channel parameters are known.

---

## Commitment 1 — HTLC Add

Alice sends 0.2 BTC to Bob via HTLC. Payment hash H, CLTV timeout T=100.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer

    Alice->>Bob: update_add_htlc(id=0, 0.2 BTC, H, cltv=100)

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_1, htlcs=[offered: H, 0.2, T=100])
    SignerA-->>Alice: sig_Af(commitment_B_1), htlc_sigs

    Alice->>Bob: commitment_signed(sig_Af(commitment_B_1), [sig_Af(htlc_success_tx)])

    Bob->>SignerB: ValidateCommitmentTx(commitment_B_1, sig_Af)
    SignerB-->>Bob: OK

    Bob->>SignerB: RevokeCommitmentTx(0)
    SignerB-->>Bob: bs0, bp2

    Bob->>Alice: revoke_and_ack(bs0, bp2)

    Bob->>SignerB: SignRemoteCommitmentTx(commitment_A_1, htlcs=[offered: H, 0.2, T=100])
    SignerB-->>Bob: sig_Bf(commitment_A_1), htlc_sigs

    Bob->>Alice: commitment_signed(sig_Bf(commitment_A_1), [sig_Bf(htlc_timeout_tx)])

    Alice->>SignerA: ValidateRevocation(0, bs0)
    SignerA-->>Alice: OK

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_1, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: RevokeCommitmentTx(0)
    SignerA-->>Alice: as0, ap2

    Alice->>Bob: revoke_and_ack(as0, ap2)

    Note over Alice, Bob: Commitment 1 established<br/>Commitment 0 revoked<br/>Alice: 0.8 + 0.2 HTLC | Bob: 0.0
```

### Signer Calls — Commitment 1

**Alice (initiator) — 4 calls:**

| Call | When | Purpose |
|------|------|---------|
| SignRemoteCommitmentTx(commitment_B_1) | After update_add_htlc, before commitment_signed | Sign Bob's new commitment with HTLC |
| ValidateRevocation(0, bs0) | After revoke_and_ack received | Verify Bob revoked commitment 0 |
| ValidateCommitmentTx(commitment_A_1, sig_Bf) | After commitment_signed received | Validate our new commitment with HTLC |
| RevokeCommitmentTx(0) | Before revoke_and_ack sent | Get as0 to revoke our commitment 0 |

**Bob (responder) — 3 calls:**

| Call | When | Purpose |
|------|------|---------|
| ValidateCommitmentTx(commitment_B_1, sig_Af) | After commitment_signed received | Validate our new commitment with HTLC |
| RevokeCommitmentTx(0) | Before revoke_and_ack sent | Get bs0 to revoke our commitment 0 |
| SignRemoteCommitmentTx(commitment_A_1) | Before commitment_signed sent | Sign Alice's new commitment with HTLC |

---

## Commitment 2 — HTLC Settlement

Bob reveals the preimage, claiming the 0.2 BTC. HTLC removed from both commitments.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer

    Bob->>Alice: update_fulfill_htlc(id=0, P)

    Bob->>SignerB: SignRemoteCommitmentTx(commitment_A_2, htlcs=[])
    SignerB-->>Bob: sig_Bf(commitment_A_2)

    Bob->>Alice: commitment_signed(sig_Bf(commitment_A_2))

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_2, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: RevokeCommitmentTx(1)
    SignerA-->>Alice: as1, ap3

    Alice->>Bob: revoke_and_ack(as1, ap3)

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_2, htlcs=[])
    SignerA-->>Alice: sig_Af(commitment_B_2)

    Alice->>Bob: commitment_signed(sig_Af(commitment_B_2))

    Bob->>SignerB: ValidateRevocation(1, as1)
    SignerB-->>Bob: OK

    Bob->>SignerB: ValidateCommitmentTx(commitment_B_2, sig_Af)
    SignerB-->>Bob: OK

    Bob->>SignerB: RevokeCommitmentTx(1)
    SignerB-->>Bob: bs1, bp3

    Bob->>Alice: revoke_and_ack(bs1, bp3)

    Note over Alice, Bob: Commitment 2 established<br/>Commitment 1 revoked<br/>Alice: 0.8 BTC | Bob: 0.2 BTC
```

### Signer Calls — Commitment 2

**Bob (initiator) — 4 calls:**

| Call | When | Purpose |
|------|------|---------|
| SignRemoteCommitmentTx(commitment_A_2) | After update_fulfill_htlc, before commitment_signed | Sign Alice's new commitment, HTLC removed |
| ValidateRevocation(1, as1) | After revoke_and_ack received | Verify Alice revoked commitment 1 |
| ValidateCommitmentTx(commitment_B_2, sig_Af) | After commitment_signed received | Validate our new commitment, HTLC removed |
| RevokeCommitmentTx(1) | Before revoke_and_ack sent | Get bs1 to revoke our commitment 1 |

**Alice (responder) — 3 calls:**

| Call | When | Purpose |
|------|------|---------|
| ValidateCommitmentTx(commitment_A_2, sig_Bf) | After commitment_signed received | Validate our new commitment, HTLC removed |
| RevokeCommitmentTx(1) | Before revoke_and_ack sent | Get as1 to revoke our commitment 1 |
| SignRemoteCommitmentTx(commitment_B_2) | Before commitment_signed sent | Sign Bob's new commitment, HTLC removed |

---

## Cooperative Close

Both sides agree to close the channel. No more HTLCs pending.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer
    participant Bitcoin

    Alice->>Bob: shutdown(scriptpubkey_A)
    Bob->>Alice: shutdown(scriptpubkey_B)

    Alice->>SignerA: SignMutualCloseTx(close_tx)
    SignerA-->>Alice: sig_Af(close_tx)

    Alice->>Bob: closing_signed(fee_satoshis, sig_Af(close_tx))

    Bob->>SignerB: SignMutualCloseTx(close_tx)
    SignerB-->>Bob: sig_Bf(close_tx)

    Bob->>Alice: closing_signed(fee_satoshis, sig_Bf(close_tx))

    Alice->>Bitcoin: broadcast close tx

    Bitcoin-->>Alice: confirmed
    Bitcoin-->>Bob: confirmed

    Note over Alice, Bob: Channel closed cooperatively
```

### Signer Calls — Cooperative Close

**Alice — 1 call:**

| Call | When | Purpose |
|------|------|---------|
| SignMutualCloseTx(close_tx) | Before closing_signed sent | Sign the cooperative close transaction |

**Bob — 1 call:**

| Call | When | Purpose |
|------|------|---------|
| SignMutualCloseTx(close_tx) | Before closing_signed sent | Sign the cooperative close transaction |
