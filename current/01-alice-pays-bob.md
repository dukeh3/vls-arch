# Scenario 01 — Alice Pays Bob (VLS Signer — Current/Separate Calls)

Same scenario as `../reference/01-alice-pays-bob.md`, but showing VLS signer calls for both Alice and Bob. This is the **current** version reflecting how VLS works today — each API call is separate (no batching).

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

    Alice->>SignerA: NewChannel
    SignerA-->>Alice: channel_id

    Alice->>SignerA: GetChannelBasepoints
    SignerA-->>Alice: Af, Ar, Ap, Ad, Ah

    Alice->>SignerA: GetPerCommitmentPoint(0)
    SignerA-->>Alice: ap0

    Alice->>Bob: open_channel
    Note over Alice, Bob: funding_satoshis=1.0 BTC<br/>Af, Ar, Ap, Ad, Ah, ap0

    Bob->>SignerB: NewChannel
    SignerB-->>Bob: channel_id

    Bob->>SignerB: GetChannelBasepoints
    SignerB-->>Bob: Bf, Br, Bp, Bd, Bh

    Bob->>SignerB: GetPerCommitmentPoint(0)
    SignerB-->>Bob: bp0

    Bob->>Alice: accept_channel
    Note over Alice, Bob: Bf, Br, Bp, Bd, Bh, bp0

    Alice->>SignerA: SetupChannel(is_outbound=true)
    Note over Alice, SignerA: Both sides' params now known
    SignerA-->>Alice: OK

    Note over Alice: Creates funding tx (unsigned)<br/>1.0 BTC to 2-of-2(Af, Bf)

    Alice->>SignerA: SignRemoteCommitmentTx(commitment_B_0)
    SignerA-->>Alice: sig_Af(commitment_B_0)

    Alice->>Bob: funding_created
    Note over Alice, Bob: funding_txid, funding_output_index<br/>sig_Af(commitment_B_0)

    Bob->>SignerB: SetupChannel(is_outbound=false)
    Note over Bob, SignerB: Both sides' params now known
    SignerB-->>Bob: OK

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

    Alice->>SignerA: CheckOutpoint
    SignerA-->>Alice: OK

    Alice->>SignerA: LockOutpoint
    SignerA-->>Alice: OK

    Alice->>SignerA: GetPerCommitmentPoint(1)
    SignerA-->>Alice: ap1

    Alice->>Bob: channel_ready
    Note over Alice, Bob: next_per_commitment_point: ap1

    Bob->>SignerB: CheckOutpoint
    SignerB-->>Bob: OK

    Bob->>SignerB: LockOutpoint
    SignerB-->>Bob: OK

    Bob->>SignerB: GetPerCommitmentPoint(1)
    SignerB-->>Bob: bp1

    Bob->>Alice: channel_ready
    Note over Alice, Bob: next_per_commitment_point: bp1

    Note over Alice, Bob: Commitment 0 established<br/>Alice: 1.0 BTC | Bob: 0.0 BTC
```

### Signer Calls Summary

**Alice (funder) — 9 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | NewChannel | Before open_channel | Register new channel with signer |
| 2 | GetChannelBasepoints | Before open_channel | Get funding key and basepoints |
| 3 | GetPerCommitmentPoint(0) | Before open_channel | Get first per-commitment point |
| 4 | SetupChannel(is_outbound=true) | After accept_channel received | Register counterparty params |
| 5 | SignRemoteCommitmentTx(commitment_B_0) | Before funding_created | Sign Bob's commitment |
| 6 | ValidateCommitmentTx(commitment_A_0, sig_Bf) | After funding_signed received | Validate Bob's sig on our commitment |
| 7 | SignWithdrawal(funding_tx) | Before broadcast | Sign the funding tx inputs |
| 8 | CheckOutpoint | After funding confirmed | Verify the funding outpoint |
| 9 | LockOutpoint | After funding confirmed | Lock channel to this outpoint |
| 10 | GetPerCommitmentPoint(1) | After funding confirmed | Get ap1 for channel_ready |

**Bob (non-funder) — 8 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | NewChannel | After open_channel received | Register new channel with signer |
| 2 | GetChannelBasepoints | After open_channel received | Get funding key and basepoints |
| 3 | GetPerCommitmentPoint(0) | After open_channel received | Get first per-commitment point |
| 4 | SetupChannel(is_outbound=false) | After funding_created received | Register counterparty params |
| 5 | ValidateCommitmentTx(commitment_B_0, sig_Af) | After funding_created received | Validate Alice's sig on our commitment |
| 6 | SignRemoteCommitmentTx(commitment_A_0) | After funding_created received | Sign Alice's commitment |
| 7 | CheckOutpoint | After funding confirmed | Verify the funding outpoint |
| 8 | LockOutpoint | After funding confirmed | Lock channel to this outpoint |
| 9 | GetPerCommitmentPoint(1) | After funding confirmed | Get bp1 for channel_ready |

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

**Alice (initiator) — 4 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | SignRemoteCommitmentTx(commitment_B_1) | Before commitment_signed sent | Sign Bob's new commitment with HTLC |
| 2 | ValidateRevocation(0, bs0) | After revoke_and_ack received | Verify Bob revoked commitment 0 |
| 3 | ValidateCommitmentTx(commitment_A_1, sig_Bf) | After commitment_signed received | Validate our new commitment with HTLC |
| 4 | RevokeCommitmentTx(0) | Before revoke_and_ack sent | Get as0 to revoke our commitment 0 |

**Bob (responder) — 3 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | ValidateCommitmentTx(commitment_B_1, sig_Af) | After commitment_signed received | Validate our new commitment with HTLC |
| 2 | RevokeCommitmentTx(0) | Before revoke_and_ack sent | Get bs0 to revoke our commitment 0 |
| 3 | SignRemoteCommitmentTx(commitment_A_1) | Before commitment_signed sent | Sign Alice's new commitment with HTLC |

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

**Bob (initiator) — 4 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | SignRemoteCommitmentTx(commitment_A_2) | Before commitment_signed sent | Sign Alice's new commitment, HTLC removed |
| 2 | ValidateRevocation(1, as1) | After revoke_and_ack received | Verify Alice revoked commitment 1 |
| 3 | ValidateCommitmentTx(commitment_B_2, sig_Af) | After commitment_signed received | Validate our new commitment, HTLC removed |
| 4 | RevokeCommitmentTx(1) | Before revoke_and_ack sent | Get bs1 to revoke our commitment 1 |

**Alice (responder) — 3 separate calls:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | ValidateCommitmentTx(commitment_A_2, sig_Bf) | After commitment_signed received | Validate our new commitment, HTLC removed |
| 2 | RevokeCommitmentTx(1) | Before revoke_and_ack sent | Get as1 to revoke our commitment 1 |
| 3 | SignRemoteCommitmentTx(commitment_B_2) | Before commitment_signed sent | Sign Bob's new commitment, HTLC removed |

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

    Alice->>Bob: shutdown
    Note over Alice, Bob: scriptpubkey_A

    Bob->>Alice: shutdown
    Note over Alice, Bob: scriptpubkey_B

    Alice->>SignerA: SignMutualCloseTx(close_tx)
    SignerA-->>Alice: sig_Af(close_tx)

    Alice->>Bob: closing_signed
    Note over Alice, Bob: fee_satoshis, sig_Af(close_tx)

    Bob->>SignerB: SignMutualCloseTx(close_tx)
    SignerB-->>Bob: sig_Bf(close_tx)

    Bob->>Alice: closing_signed
    Note over Alice, Bob: fee_satoshis, sig_Bf(close_tx)

    Alice->>Bitcoin: broadcast close tx
    Note over Bitcoin: close_tx spends funding output<br/>0.8 BTC → scriptpubkey_A<br/>0.2 BTC → scriptpubkey_B

    Bitcoin-->>Alice: confirmed
    Bitcoin-->>Bob: confirmed

    Note over Alice, Bob: Channel closed cooperatively
```

### Signer Calls — Cooperative Close

**Alice — 1 call:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | SignMutualCloseTx(close_tx) | Before closing_signed sent | Sign the cooperative close transaction |

**Bob — 1 call:**

| # | Call | When | Purpose |
|---|------|------|---------|
| 1 | SignMutualCloseTx(close_tx) | Before closing_signed sent | Sign the cooperative close transaction |
