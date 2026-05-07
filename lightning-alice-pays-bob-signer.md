# Lightning Messages — Alice Pays Bob (Both Have VLS Signers)

Same scenario as `lightning-alice-pays-bob-messages.md`, but showing VLS signer calls for both Alice and Bob.

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

    Alice->>SignerA: NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) + SetupChannel(is_outbound=true)
    SignerA-->>Alice: Af, Ar, Ap, Ad, Ah, ap0

    Alice->>Bob: open_channel
    Note over Alice, Bob: funding_satoshis=1.0 BTC<br/>Af, Ar, Ap, Ad, Ah, ap0

    Bob->>SignerB: NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) + SetupChannel(is_outbound=false)
    SignerB-->>Bob: Bf, Br, Bp, Bd, Bh, bp0

    Bob->>Alice: accept_channel
    Note over Alice, Bob: Bf, Br, Bp, Bd, Bh, bp0

    Note over Alice: Creates funding tx (unsigned)<br/>1.0 BTC to 2-of-2(Af, Bf)

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_0)
    SignerA-->>Alice: sig_Af(commitment_B_0)

    Alice->>Bob: funding_created
    Note over Alice, Bob: funding_txid, funding_output_index<br/>sig_Af(commitment_B_0)

    Bob->>SignerB: ValidateCommitmentTx(commitment_B_0, sig_Af)
    SignerB-->>Bob: OK

    Bob->>SignerB: SignRemoteCommitmentTx(commitment_A_0)
    SignerB-->>Bob: sig_Bf(commitment_A_0)

    Bob->>Alice: funding_signed
    Note over Alice, Bob: sig_Bf(commitment_A_0)

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_0, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: SignWithdrawal(funding_tx)
    SignerA-->>Alice: signed funding tx

    Alice->>Bitcoin: broadcast funding tx

    Bitcoin-->>Alice: confirmed
    Bitcoin-->>Bob: confirmed

    Alice->>SignerA: CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1)
    SignerA-->>Alice: ap1

    Alice->>Bob: channel_ready
    Note over Alice, Bob: next_per_commitment_point: ap1

    Bob->>SignerB: CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1)
    SignerB-->>Bob: bp1

    Bob->>Alice: channel_ready
    Note over Alice, Bob: next_per_commitment_point: bp1

    Note over Alice, Bob: Commitment 0 established<br/>Alice: 1.0 BTC | Bob: 0.0 BTC
```

### Signer Calls Summary

**Alice (funder) — 5 calls:**

| Call | When | Purpose |
|------|------|---------|
| NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) + SetupChannel | Before open_channel sent | Get keys, register channel |
| SignRemoteCommitmentTx(commitment_B_0) | Before funding_created sent | Sign Bob's commitment for him |
| ValidateCommitmentTx(commitment_A_0, sig_Bf) | After funding_signed received | Validate Bob's sig on our commitment |
| SignWithdrawal(funding_tx) | Before broadcast | Sign the funding tx inputs |
| CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1) | After funding tx confirmed | Lock channel, get ap1 for channel_ready |

**Bob (non-funder) — 4 calls:**

| Call | When | Purpose |
|------|------|---------|
| NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0) + SetupChannel | After open_channel received | Get keys, register channel |
| ValidateCommitmentTx(commitment_B_0, sig_Af) + SignRemoteCommitmentTx(commitment_A_0) | After funding_created received | Validate Alice's sig, sign her commitment |
| CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1) | After funding tx confirmed | Lock channel, get bp1 for channel_ready |

Alice has 2 extra calls: `SignRemoteCommitmentTx` (she signs first, before funding_created) and `SignWithdrawal` (she owns the funding tx).

---

## Commitment 1 — HTLC Add

Alice sends 0.2 BTC to Bob via HTLC. Payment hash H, CLTV timeout T=100.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer

    Alice->>Bob: update_add_htlc(H, 0.2 BTC, cltv=100)

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_1, htlcs=[offered: H, 0.2, T=100])
    SignerA-->>Alice: sig_Af(commitment_B_1), htlc_sigs

    Alice->>Bob: commitment_signed
    Note over Alice, Bob: sig_Af(commitment_B_1), htlc_sigs

    Bob->>SignerB: ValidateCommitmentTx(commitment_B_1, sig_Af)
    SignerB-->>Bob: OK

    Bob->>SignerB: RevokeCommitmentTx(0)
    SignerB-->>Bob: bs0, bp2

    Bob->>Alice: revoke_and_ack
    Note over Alice, Bob: bs0, bp2

    Bob->>SignerB: SignRemoteCommitmentTx(commitment_A_1, htlcs=[offered: H, 0.2, T=100])
    SignerB-->>Bob: sig_Bf(commitment_A_1), htlc_sigs

    Bob->>Alice: commitment_signed
    Note over Alice, Bob: sig_Bf(commitment_A_1), htlc_sigs

    Alice->>SignerA: ValidateRevocation(0, bs0)
    SignerA-->>Alice: OK

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_1, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: RevokeCommitmentTx(0)
    SignerA-->>Alice: as0, ap2

    Alice->>Bob: revoke_and_ack
    Note over Alice, Bob: as0, ap2

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

    Bob->>Alice: update_fulfill_htlc(H, preimage=P)

    Bob->>SignerB: SignRemoteCommitmentTx(commitment_A_2, htlcs=[])
    SignerB-->>Bob: sig_Bf(commitment_A_2)

    Bob->>Alice: commitment_signed
    Note over Alice, Bob: sig_Bf(commitment_A_2)

    Alice->>SignerA: ValidateCommitmentTx(commitment_A_2, sig_Bf)
    SignerA-->>Alice: OK

    Alice->>SignerA: RevokeCommitmentTx(1)
    SignerA-->>Alice: as1, ap3

    Alice->>Bob: revoke_and_ack
    Note over Alice, Bob: as1, ap3

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_2, htlcs=[])
    SignerA-->>Alice: sig_Af(commitment_B_2)

    Alice->>Bob: commitment_signed
    Note over Alice, Bob: sig_Af(commitment_B_2)

    Bob->>SignerB: ValidateRevocation(1, as1)
    SignerB-->>Bob: OK

    Bob->>SignerB: ValidateCommitmentTx(commitment_B_2, sig_Af)
    SignerB-->>Bob: OK

    Bob->>SignerB: RevokeCommitmentTx(1)
    SignerB-->>Bob: bs1, bp3

    Bob->>Alice: revoke_and_ack
    Note over Alice, Bob: bs1, bp3

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
