# Lightning Messages — Alice Pays Bob

Sequence diagrams for each state transition in the "Alice pays Bob" drawio diagram. Alice funds the channel with 1.0 BTC, pays Bob 0.2 BTC via HTLC.

See also: [Key Derivation](lightning-key-derivation.md) for how the keys referenced here are calculated.

---

## Commitment 0

Alice funds a 1.0 BTC channel. After this exchange, both sides hold commitment 0.

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    participant Bitcoin

    Alice->>Bob: open_channel
    Note right of Alice: chain_hash, funding_satoshis=1.0 BTC<br/>push_msat=0, to_self_delay=10<br/>funding_pubkey: Af<br/>revocation_basepoint: Ar<br/>payment_basepoint: Ap<br/>delayed_payment_basepoint: Ad<br/>htlc_basepoint: Ah<br/>first_per_commitment_point: ap0

    Bob->>Alice: accept_channel
    Note right of Bob: to_self_delay=10<br/>funding_pubkey: Bf<br/>revocation_basepoint: Br<br/>payment_basepoint: Bp<br/>delayed_payment_basepoint: Bd<br/>htlc_basepoint: Bh<br/>first_per_commitment_point: bp0

    Note over Alice: Creates funding tx (unsigned)<br/>1.0 BTC to 2-of-2(Af, Bf)

    Note over Alice: Builds commitment_B_0<br/>and signs it with her funding key Af

    Alice->>Bob: funding_created
    Note right of Alice: funding_txid, funding_output_index<br/>sig_Af(commitment_B_0)

    Note over Bob: Builds commitment_A_0<br/>and signs it with his funding key Bf

    Bob->>Alice: funding_signed
    Note right of Bob: sig_Bf(commitment_A_0)

    Note over Alice: Has sig_Bf for commitment_A_0<br/>Safe to broadcast funding tx

    Alice->>Bitcoin: broadcast funding tx
    Note over Alice, Bitcoin: 1.0 BTC to 2-of-2(Af, Bf)

    Bitcoin-->>Alice: confirmed
    Bitcoin-->>Bob: confirmed

    Alice->>Bob: channel_ready
    Note right of Alice: next_per_commitment_point: ap1

    Bob->>Alice: channel_ready
    Note right of Bob: next_per_commitment_point: bp1

    Note over Alice, Bob: Commitment 0 established<br/>Alice: 1.0 BTC | Bob: 0.0 BTC
```

### Commitment 0 Transactions

After channel open, each side holds:

**commitment_A_0** (Alice holds, signed by Bob with Bf):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:  1.0 BTC — rp(Br, ap0) | dp(Ad, ap0) + dt=10
  to_remote: 0.0 BTC — Bp (not created, below dust)
```

**commitment_B_0** (Bob holds, signed by Alice with Af):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:  0.0 BTC — rp(Ar, bp0) | dp(Bd, bp0) + dt=10 (not created, below dust)
  to_remote: 1.0 BTC — Ap
```

---

## Commitment 1 — HTLC Add

Alice sends 0.2 BTC to Bob via HTLC. Payment hash H, CLTV timeout T=100.

```mermaid
sequenceDiagram
    participant Alice
    participant Bob

    Alice->>Bob: update_add_htlc
    Note over Alice, Bob: hash=H, amount=0.2 BTC, cltv_expiry=100

    Alice->>Bob: commitment_signed
    Note over Alice, Bob: sig_Af(commitment_B_1), htlc_sigs

    Bob->>Alice: revoke_and_ack
    Note over Alice, Bob: per_commitment_secret: bs0<br/>next_per_commitment_point: bp2

    Bob->>Alice: commitment_signed
    Note over Alice, Bob: sig_Bf(commitment_A_1), htlc_sigs

    Alice->>Bob: revoke_and_ack
    Note over Alice, Bob: per_commitment_secret: as0<br/>next_per_commitment_point: ap2

    Note over Alice, Bob: Commitment 1 established<br/>Commitment 0 revoked (as0, bs0 revealed)<br/>Alice: 0.8 + 0.2 HTLC | Bob: 0.0
```

### Commitment 1 Transactions

**commitment_A_1** (Alice holds, signed by Bob with Bf):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:      0.8 BTC — rp(Br, ap1) | dp(Ad, ap1) + dt=10
  htlc_offered:  0.2 BTC — rp(Br, ap1) | sig_Af+sig_Bf (HTLC-timeout, after T=100) | Bh(bp1) + H(P)
  to_remote:     0.0 BTC — Bp (not created, below dust)
```

**commitment_B_1** (Bob holds, signed by Alice with Af):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:      0.0 BTC — rp(Ar, bp1) | dp(Bd, bp1) + dt=10 (not created, below dust)
  htlc_received: 0.2 BTC — rp(Ar, bp1) | sig_Af+sig_Bf (HTLC-success, with preimage) | Ah(ap1) + T=100
  to_remote:     0.8 BTC — Ap
```

The HTLC output has three spend paths:
1. **Revocation** — counterparty takes all if revoked commitment is broadcast
2. **2nd-stage tx** — requires both signatures, goes to a timeout or success transaction with CSV delay
3. **Direct claim** — preimage (for received) or timeout (for offered)

---

## Commitment 2 — HTLC Settlement

Bob reveals the preimage, claiming the 0.2 BTC.

```mermaid
sequenceDiagram
    participant Alice
    participant Bob

    Bob->>Alice: update_fulfill_htlc
    Note over Alice, Bob: hash=H, preimage=P

    Bob->>Alice: commitment_signed
    Note over Alice, Bob: sig_Bf(commitment_A_2)

    Alice->>Bob: revoke_and_ack
    Note over Alice, Bob: per_commitment_secret: as1<br/>next_per_commitment_point: ap3

    Alice->>Bob: commitment_signed
    Note over Alice, Bob: sig_Af(commitment_B_2)

    Bob->>Alice: revoke_and_ack
    Note over Alice, Bob: per_commitment_secret: bs1<br/>next_per_commitment_point: bp3

    Note over Alice, Bob: Commitment 2 established<br/>Commitment 1 revoked (as1, bs1 revealed)<br/>Alice: 0.8 BTC | Bob: 0.2 BTC
```

### Commitment 2 Transactions

**commitment_A_2** (Alice holds, signed by Bob with Bf):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:  0.8 BTC — rp(Br, ap2) | dp(Ad, ap2) + dt=10
  to_remote: 0.2 BTC — Bp
```

**commitment_B_2** (Bob holds, signed by Alice with Af):
```
Input:  funding_outpoint (requires sig_Af + sig_Bf)
Outputs:
  to_local:  0.2 BTC — rp(Ar, bp2) | dp(Bd, bp2) + dt=10
  to_remote: 0.8 BTC — Ap
```
