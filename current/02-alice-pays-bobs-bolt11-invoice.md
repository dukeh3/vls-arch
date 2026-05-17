# Scenario 02 — Alice Pays Bob's Invoice (VLS Signer — Current/Separate Calls)

Alice and Bob already have an open channel (established in [Scenario 01](01-alice-pays-bob.md)). Bob creates a BOLT11 invoice, Alice pays it. This scenario shows the VLS calls involved in **invoice creation**, **payment preapproval**, and **HTLC settlement** — focusing on how the signer participates at each stage.

For notation (`Af`, `Ar`, `ap0`, `rp(...)`, etc.) see [Key Derivation](../reference/lightning-key-derivation.md#notation-legend).

### Source References

Wire message definitions — `vls-protocol/src/msgs.rs`:

| Call | Message struct | Line |
|------|---------------|------|
| SignInvoice | `SignInvoice` | [msgs.rs:?] |
| PreapproveInvoice | `PreapproveInvoice` | [msgs.rs:?] |
| SignRemoteCommitmentTx | `SignRemoteCommitmentTx` | [msgs.rs:353](../validating-lightning-signer/vls-protocol/src/msgs.rs#L353) |
| ValidateCommitmentTx | `ValidateCommitmentTx` | [msgs.rs:566](../validating-lightning-signer/vls-protocol/src/msgs.rs#L566) |
| RevokeCommitmentTx | `RevokeCommitmentTx` | [msgs.rs:644](../validating-lightning-signer/vls-protocol/src/msgs.rs#L644) |
| ValidateRevocation | `ValidateRevocation` | [msgs.rs:587](../validating-lightning-signer/vls-protocol/src/msgs.rs#L587) |

---

## Prerequisites

- Channel from Scenario 01 is open and at Commitment 2 (Alice: 0.8 BTC, Bob: 0.2 BTC)
- Bob wants to receive 0.1 BTC from Alice

---

## Phase 1 — Bob Creates Invoice

Bob's node creates a BOLT11 invoice and asks the signer to sign it.

```mermaid
sequenceDiagram
    participant SignerB as Bob Signer
    participant Bob
    participant App as Bob's App

    App->>Bob: "Create invoice for 0.1 BTC"

    Note over Bob: Derives payment_hash from<br/>inbound_payment_key (deterministic):<br/>H = HMAC(inbound_payment_key, metadata)

    Note over Bob: Builds invoice:<br/>lnbc100m...  (amount, H, expiry, route hints)

    Bob->>SignerB: SignInvoice(<br/>u5bytes=<invoice_data>,<br/>hrp="lnbc100m")
    SignerB-->>Bob: signature (65 bytes, recoverable)

    Note over Bob: Appends signature to invoice.<br/>Payee node_id is recoverable from sig.

    Bob-->>App: lnbc100m1pj....<bech32_invoice>
```

### What the signer does on `SignInvoice`

The signer signs the invoice using `node_secret` (`m/0'`). The signature allows any payer to recover `node_id` (Bob's public key) from the invoice — this is how the payer knows where to route the payment.

**The signer could also:**
- Record the `payment_hash` from the invoice (for later verification when an HTLC arrives)
- Check invoice amount against policy limits
- Check that the invoice hasn't already expired

**Currently:** The signer signs without policy checks on invoice creation. It trusts the node to create valid invoices.

### VLS Calls — Invoice Creation

| # | Call | Input | Output | Policy checks |
|---|------|-------|--------|---------------|
| 1 | SignInvoice | `u5bytes` (base32 invoice data), `hrp` ("lnbc100m") | 65-byte recoverable ECDSA signature | None currently |

---

## Phase 2 — Alice Receives Invoice and Preapproves Payment

Alice receives the invoice out-of-band (QR code, message, etc.) and her node asks the signer to preapprove the payment before routing.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant App as Alice's App

    App->>Alice: "Pay this invoice: lnbc100m1pj..."

    Alice->>SignerA: PreapproveInvoice(<br/>invstring="lnbc100m1pj...")
    SignerA-->>Alice: approved=true

    Note over Alice: Finds route to Bob's node_id<br/>(recovered from invoice signature)
```

### What the signer does on `PreapproveInvoice`

The signer parses the invoice and checks:

| Check | Policy rule | What it prevents |
|-------|-------------|------------------|
| Invoice not expired | `policy-invoice-not-expired` (rule 59) | Paying a stale invoice that can't be settled |
| Amount within limits | `policy-commitment-payment-invoiced` (rule 58) | Overpaying or exceeding velocity limits |
| Destination allowed | (allowlist, if configured) | Payments to unknown/banned nodes |

If the signer rejects, the node must not initiate the HTLC. This is the signer's opportunity to prevent the payment **before** any channel state changes.

### VLS Calls — Payment Preapproval

| # | Call | Input | Output | Policy checks |
|---|------|-------|--------|---------------|
| 1 | PreapproveInvoice | `invstring` (full BOLT11 string) | `approved` (bool) | Invoice expiry, amount limits, velocity, destination |

---

## Phase 3 — HTLC Add (Commitment 3)

Alice sends the payment as an HTLC. This is structurally identical to Commitment 1 in Scenario 01, with different balances.

Starting state: Alice 0.8 BTC, Bob 0.2 BTC (Commitment 2)

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer

    Alice->>Bob: update_add_htlc(id=1, 0.1 BTC, H, cltv=150)

    Alice->>SignerA: SignRemoteCommitmentTx(<br/>tx=commitment_B_3, psbt,<br/>remote_funding_key=Bf,<br/>remote_per_commitment_point=bp3,<br/>commitment_number=3, feerate,<br/>htlcs=[offered: H, 0.1 BTC, T=150])
    SignerA-->>Alice: sig_Af(commitment_B_3)

    Alice->>Bob: commitment_signed(sig_Af(commitment_B_3), [sig_Af(htlc_success_tx)])

    Bob->>SignerB: ValidateCommitmentTx(<br/>tx=commitment_B_3, psbt,<br/>commitment_number=3, feerate,<br/>htlcs=[received: H, 0.1 BTC, T=150],<br/>signature=sig_Af, htlc_signatures)
    SignerB-->>Bob: next_per_commitment_point=bp4

    Bob->>SignerB: RevokeCommitmentTx(commitment_number=2)
    SignerB-->>Bob: old_commitment_secret=bs2, next_per_commitment_point=bp4

    Bob->>SignerB: SignRemoteCommitmentTx(<br/>tx=commitment_A_3, psbt,<br/>remote_funding_key=Af,<br/>remote_per_commitment_point=ap3,<br/>commitment_number=3, feerate,<br/>htlcs=[offered: H, 0.1 BTC, T=150])
    SignerB-->>Bob: sig_Bf(commitment_A_3)

    Bob->>Alice: revoke_and_ack(bs2, bp4)
    Bob->>Alice: commitment_signed(sig_Bf(commitment_A_3), [sig_Bf(htlc_timeout_tx)])

    Alice->>SignerA: ValidateRevocation(<br/>commitment_number=2,<br/>commitment_secret=bs2)
    SignerA-->>Alice: OK

    Alice->>SignerA: ValidateCommitmentTx(<br/>tx=commitment_A_3, psbt,<br/>commitment_number=3, feerate,<br/>htlcs=[offered: H, 0.1 BTC, T=150],<br/>signature=sig_Bf, htlc_signatures)
    SignerA-->>Alice: next_per_commitment_point=ap4

    Alice->>SignerA: RevokeCommitmentTx(commitment_number=2)
    SignerA-->>Alice: old_commitment_secret=as2, next_per_commitment_point=ap4

    Alice->>Bob: revoke_and_ack(as2, ap4)

    Bob->>SignerB: ValidateRevocation(<br/>commitment_number=2,<br/>commitment_secret=as2)
    SignerB-->>Bob: OK

    Note over Alice, Bob: Commitment 3 established<br/>Alice: 0.7 + 0.1 HTLC | Bob: 0.2
```

### Signer Calls — Commitment 3 (HTLC Add)

**Alice (sender) — 4 calls:**

| # | Call | Returns |
|---|------|---------|
| 1 | SignRemoteCommitmentTx | sig_Af(commitment_B_3) |
| 2 | ValidateRevocation | — (stores bs2) |
| 3 | ValidateCommitmentTx | next_pcp=ap4 |
| 4 | RevokeCommitmentTx | as2, next_pcp=ap4 |

**Bob (receiver) — 4 calls:**

| # | Call | Returns |
|---|------|---------|
| 1 | ValidateCommitmentTx | next_pcp=bp4 |
| 2 | RevokeCommitmentTx | bs2, next_pcp=bp4 |
| 3 | SignRemoteCommitmentTx | sig_Bf(commitment_A_3) |
| 4 | ValidateRevocation | — (stores as2) |

---

## Phase 4 — HTLC Settlement (Commitment 4)

Bob knows the preimage P (he generated H = SHA256(P) when creating the invoice). He reveals it to settle the HTLC.

```mermaid
sequenceDiagram
    participant SignerA as Alice Signer
    participant Alice
    participant Bob
    participant SignerB as Bob Signer

    Note over Bob: Knows preimage P where H = SHA256(P)

    Bob->>Alice: update_fulfill_htlc(id=1, P)

    Bob->>SignerB: SignRemoteCommitmentTx(<br/>tx=commitment_A_4, psbt,<br/>remote_funding_key=Af,<br/>remote_per_commitment_point=ap4,<br/>commitment_number=4, feerate,<br/>htlcs=[])
    SignerB-->>Bob: sig_Bf(commitment_A_4)

    Bob->>Alice: commitment_signed(sig_Bf(commitment_A_4))

    Alice->>SignerA: ValidateCommitmentTx(<br/>tx=commitment_A_4, psbt,<br/>commitment_number=4, feerate,<br/>htlcs=[],<br/>signature=sig_Bf, htlc_signatures=[])
    SignerA-->>Alice: next_per_commitment_point=ap5

    Alice->>SignerA: RevokeCommitmentTx(commitment_number=3)
    SignerA-->>Alice: old_commitment_secret=as3, next_per_commitment_point=ap5

    Alice->>SignerA: SignRemoteCommitmentTx(<br/>tx=commitment_B_4, psbt,<br/>remote_funding_key=Bf,<br/>remote_per_commitment_point=bp4,<br/>commitment_number=4, feerate,<br/>htlcs=[])
    SignerA-->>Alice: sig_Af(commitment_B_4)

    Alice->>Bob: revoke_and_ack(as3, ap5)
    Alice->>Bob: commitment_signed(sig_Af(commitment_B_4))

    Bob->>SignerB: ValidateRevocation(<br/>commitment_number=3,<br/>commitment_secret=as3)
    SignerB-->>Bob: OK

    Bob->>SignerB: ValidateCommitmentTx(<br/>tx=commitment_B_4, psbt,<br/>commitment_number=4, feerate,<br/>htlcs=[],<br/>signature=sig_Af, htlc_signatures=[])
    SignerB-->>Bob: next_per_commitment_point=bp5

    Bob->>SignerB: RevokeCommitmentTx(commitment_number=3)
    SignerB-->>Bob: old_commitment_secret=bs3, next_per_commitment_point=bp5

    Bob->>Alice: revoke_and_ack(bs3, bp5)

    Alice->>SignerA: ValidateRevocation(<br/>commitment_number=3,<br/>commitment_secret=bs3)
    SignerA-->>Alice: OK

    Note over Alice, Bob: Commitment 4 established<br/>Alice: 0.7 BTC | Bob: 0.3 BTC<br/>Invoice paid!
```

### Signer Calls — Commitment 4 (HTLC Settlement)

**Bob (initiator) — 4 calls:**

| # | Call | Returns |
|---|------|---------|
| 1 | SignRemoteCommitmentTx | sig_Bf(commitment_A_4) |
| 2 | ValidateRevocation | — (stores as3) |
| 3 | ValidateCommitmentTx | next_pcp=bp5 |
| 4 | RevokeCommitmentTx | bs3, next_pcp=bp5 |

**Alice (responder) — 4 calls:**

| # | Call | Returns |
|---|------|---------|
| 1 | ValidateCommitmentTx | next_pcp=ap5 |
| 2 | RevokeCommitmentTx | as3, next_pcp=ap5 |
| 3 | SignRemoteCommitmentTx | sig_Af(commitment_B_4) |
| 4 | ValidateRevocation | — (stores bs3) |

---

## Full Scenario Summary

### Total VLS calls (both sides)

| Phase | Alice's signer | Bob's signer |
|-------|---------------|--------------|
| 1. Invoice creation | — | 1 (SignInvoice) |
| 2. Payment preapproval | 1 (PreapproveInvoice) | — |
| 3. HTLC add (commitment 3) | 4 | 4 |
| 4. HTLC settle (commitment 4) | 4 | 4 |
| **Total** | **9** | **9** |

### What each signer verified

**Bob's signer:**
- Signed the invoice (proving Bob's node_id, enabling payment routing)
- Validated 2 new commitments (3 and 4) — checked signatures, amounts, HTLC correctness
- Revoked 2 old commitments — released secrets for penalty enforcement
- Stored 2 revocation secrets from Alice — can punish if she broadcasts old state

**Alice's signer:**
- Preapproved the payment — checked invoice validity, amount, destination, velocity
- Signed 2 counterparty commitments — produced signatures for Bob's new states
- Validated 2 new commitments — checked Bob's signatures on her states
- Revoked 2 old commitments — released secrets
- Stored 2 revocation secrets from Bob

---

## Policy Gap: Payment Hash Verification

When Bob's signer receives `ValidateCommitmentTx` with an incoming HTLC (Phase 3), it sees the `payment_hash` H. Since the signer holds the `inbound_payment_key` (derived from seed at `m/5'`), it **could** check:

> "Did I generate this payment_hash via `SignInvoice`?"

This check is documented as **Rule 37** (`policy-commitment-htlc-offered-hash-matches`) but is **not yet implemented**. If it were, the signer could reject HTLCs for invoices it never signed — preventing the node from accepting fabricated payments.

| What the signer knows | Available at |
|------------------------|-------------|
| `inbound_payment_key` | Always (derived from seed) |
| Payment hashes it signed | `SignInvoice` time |
| Incoming HTLC payment hash | `ValidateCommitmentTx` time |

The verification would connect these two moments: record H at SignInvoice, verify H at ValidateCommitmentTx.
