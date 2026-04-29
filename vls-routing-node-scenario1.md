# Routing Node with VLS — Scenario 1 Breakdown

This document breaks down the complete lifecycle of a VLS-backed routing node: opening channels, forwarding payments (HTLCs), earning routing fees, and closing channels. Every interaction between the node and signer is documented in detail.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Key Data Structures](#key-data-structures)
3. [Phase 1: Initialization](#phase-1-initialization)
4. [Phase 2: Open Channels to Peers](#phase-2-open-channels-to-peers)
5. [Phase 3: Receive and Forward an HTLC (Routing)](#phase-3-receive-and-forward-an-htlc-routing)
6. [Phase 4: HTLC Settlement (Preimage Flows Back)](#phase-4-htlc-settlement-preimage-flows-back)
7. [Phase 5: Accumulate Routing Fees](#phase-5-accumulate-routing-fees)
8. [Phase 6: Cooperative Close](#phase-6-cooperative-close)
9. [Signer-Side Validation Deep Dive](#signer-side-validation-deep-dive)
10. [Routing Fee Economics](#routing-fee-economics)
11. [Edge Cases and Error Handling](#edge-cases-and-error-handling)

---

## Architecture Overview

A routing node sits between two or more peers, forwarding payments across channels. The VLS signer holds all private keys and validates every operation.

```
                        Lightning Network
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
     ┌────┴────┐         ┌───┴───┐          ┌────┴────┐
     │  Alice  │◄───CH1──►│ OUR   │◄───CH2───►│   Bob   │
     │ (peer)  │         │ NODE  │          │ (peer)  │
     └─────────┘         └───┬───┘          └─────────┘
                             │
                     ┌───────┴───────┐
                     │  VLS SIGNER   │
                     │               │
                     │ ┌───────────┐ │
                     │ │ Channel 1 │ │  (Alice channel state)
                     │ │ Channel 2 │ │  (Bob channel state)
                     │ │ NodeState │ │  (cross-channel payment tracking)
                     │ └───────────┘ │
                     │               │
                     │  Private Keys │
                     │  Policy Engine│
                     └───────────────┘
```

**Key property:** The signer manages ALL channels under a single `NodeState`. This is what enables cross-channel HTLC correlation — the signer sees both the incoming HTLC (from Alice) and the outgoing HTLC (to Bob) and validates that the balance is correct.

---

## Key Data Structures

### Inside the Signer

```
Node
├── channels: Map<ChannelId, ChannelSlot>
│   ├── channel_1 (Alice ↔ Us)
│   │   ├── channel_value: 1,000,000 sat
│   │   ├── holder_commitment_info (our current commitment)
│   │   │   ├── offered_htlcs: [...]   ← outgoing to Alice
│   │   │   └── received_htlcs: [...]  ← incoming from Alice
│   │   └── counterparty_commitment_info
│   │       ├── offered_htlcs: [...]   ← Alice's outgoing (= our incoming)
│   │       └── received_htlcs: [...]  ← Alice's incoming (= our outgoing)
│   │
│   └── channel_2 (Bob ↔ Us)
│       ├── channel_value: 1,000,000 sat
│       ├── holder_commitment_info
│       │   ├── offered_htlcs: [...]   ← outgoing to Bob
│       │   └── received_htlcs: [...]  ← incoming from Bob
│       └── counterparty_commitment_info
│
├── state: NodeState
│   ├── payments: Map<PaymentHash, RoutedPayment>
│   │   └── RoutedPayment {
│   │       incoming: Map<ChannelId, amount_sat>   ← per-channel incoming
│   │       outgoing: Map<ChannelId, amount_sat>   ← per-channel outgoing
│   │       incoming_cltv_min: Option<u32>         ← earliest incoming timeout
│   │       outgoing_cltv_max: Option<u32>         ← latest outgoing timeout
│   │       preimage: Option<PaymentPreimage>
│   │   }
│   │
│   ├── invoices: Map<PaymentHash, PaymentState>   ← for our own payments
│   ├── issued_invoices: Map<PaymentHash, ...>     ← invoices we created
│   └── excess_amount: u64                         ← accumulated routing fees
│
└── validator: SimpleValidator
    └── policy: SimplePolicy {
        max_routing_fee_msat: u64,
        max_feerate_percentage: u8,
        cltv_delta: u32,           (default: 34)
        enforce_balance: bool,
    }
```

### HTLC Representation

```
HTLCInfo2 {
    value_sat: u64,           // HTLC amount
    payment_hash: [u8; 32],   // Correlates incoming ↔ outgoing
    cltv_expiry: u32,         // Absolute block height timeout
}
```

The `payment_hash` is the critical correlation key — the same hash appears on both the incoming and outgoing channels for a routed payment.

### Offered vs Received (Signer's Perspective)

| Commitment Type | `offered_htlcs` | `received_htlcs` |
|-----------------|-----------------|-------------------|
| **Holder** (our commitment) | HTLCs we're sending out | HTLCs we're receiving |
| **Counterparty** (their commitment) | HTLCs they're sending to us (our incoming) | HTLCs they're receiving from us (our outgoing) |

---

## Phase 1: Initialization

The routing node starts up and initializes its VLS session.

```
Node                                    Signer
 │                                        │
 │── HsmdInit(                           │
 │     chain_params=mainnet,             │
 │     hsm_wire_min_version=2,           │
 │     hsm_wire_max_version=6)       ──►│
 │                                        │
 │                                        │  Signer initializes:
 │                                        │  - Loads/generates node key
 │                                        │  - Creates NodeState
 │                                        │  - Initializes empty payments map
 │                                        │  - Sets excess_amount = 0
 │                                        │  - Negotiates protocol version
 │                                        │
 │◄── HsmdInitReplyV4(                  │
 │     hsm_version=6,                    │
 │     hsm_capabilities=[...],           │
 │     node_id=03abc...,                 │
 │     bip32=xpub...,                    │
 │     bolt12=02def...)              ────│
 │                                        │
 │  Node now knows its identity and       │
 │  can begin opening channels            │
```

---

## Phase 2: Open Channels to Peers

The routing node needs at least two channels. Let's open one to Alice and one to Bob.

### Channel 1: Alice ↔ Us (1,000,000 sat, we fund)

```
Node                                    Signer
 │                                        │
 │── NewChannel(alice_id, dbid=1) ──────►│  Register channel
 │◄── NewChannelReply ──────────────────│
 │                                        │
 │── GetChannelBasepoints(alice_id, 1) ─►│  Get our keys for this channel
 │◄── (basepoints, funding_pubkey) ────│
 │                                        │
 │── GetPerCommitmentPoint(0) ─────────►│  First commitment point
 │◄── (point_0) ───────────────────────│
 │                                        │
 │  ... negotiate with Alice ...          │
 │                                        │
 │── SetupChannel(                       │
 │     is_outbound=true,                 │  We funded this channel
 │     channel_value=1,000,000,          │
 │     push_value=0,                     │
 │     funding_txid=aaa...,              │
 │     funding_txout=0,                  │
 │     to_self_delay=144,               │
 │     remote_basepoints=alice_bp,       │
 │     remote_funding_pubkey=alice_fp,   │
 │     remote_to_self_delay=144,         │
 │     channel_type=anchors_zero_fee) ►│
 │                                        │
 │                                        │  Signer validates:
 │                                        │  ✅ contest delay in range [144, 2016]
 │                                        │  ✅ channel type is safe (AnchorsZeroFeeHtlc)
 │                                        │  ✅ no channel push
 │                                        │
 │◄── SetupChannelReply ───────────────│
 │                                        │
 │── SignRemoteCommitmentTx(0, ...) ───►│  Sign Alice's initial commitment
 │◄── SignTxReply(sig) ────────────────│  ✅ No HTLCs, correct structure
 │                                        │
 │── ValidateCommitmentTx(0, alice_sig) ►│ Validate our initial commitment
 │◄── ValidateCommitmentTxReply ───────│  ✅ funding value matches,
 │                                        │     no HTLCs, sig valid
 │                                        │
 │── SignWithdrawal(utxos, psbt) ──────►│  Sign funding transaction
 │◄── SignWithdrawalReply(signed) ─────│  ✅ SegWit inputs, outputs known,
 │                                        │     fee in range, commitment
 │                                        │     countersigned
 │  ... funding confirms ...              │
 │                                        │
 │── CheckOutpoint + LockOutpoint ─────►│  Confirm and lock funding
 │◄── (is_buried=true) ───────────────│
```

### Channel 2: Bob ↔ Us (1,000,000 sat, we fund)

Same flow as Channel 1, with `dbid=2` and Bob's parameters.

After both channels are open:

```
Signer State:
├── channels:
│   ├── CH1 (Alice): value=1,000,000, our_balance=1,000,000, their_balance=0
│   └── CH2 (Bob):   value=1,000,000, our_balance=1,000,000, their_balance=0
├── payments: {} (empty)
└── excess_amount: 0
```

---

## Phase 3: Receive and Forward an HTLC (Routing)

Alice wants to pay Bob 100,000 sat through our routing node. She sends an HTLC to us, and we forward it to Bob.

### 3.1 — Alice Sends HTLC to Us (Incoming)

Alice proposes a new commitment on Channel 1 that includes an HTLC she's offering to us.

```
Alice ──update_add_htlc(hash=H, amount=100,100 sat, cltv=800,034)──► Our Node

Alice ──commitment_signed(commitment_1, sig, htlc_sigs)──► Our Node
```

The amount is 100,100 sat because Alice includes a routing fee (100 sat).

Our node passes Alice's commitment_signed to the signer:

```
Node                                    Signer
 │                                        │
 │  Received commitment_signed from Alice │
 │  for Channel 1, commitment #1         │
 │  Contains: offered HTLC (hash=H,      │
 │            amount=100,100 sat,         │
 │            cltv_expiry=800,034)        │
 │                                        │
 │── ValidateCommitmentTx(              │
 │     CH1,                              │
 │     commitment_number=1,              │
 │     feerate=2500,                     │
 │     htlcs=[                           │
 │       { side=remote,                  │  "remote" = Alice offered to us
 │         amount=100,100 sat,           │  = our INCOMING
 │         payment_hash=H,              │
 │         cltv_expiry=800,034 }        │
 │     ],                                │
 │     signature=alice_sig,              │
 │     htlc_signatures=[alice_htlc_sig]) │
 │                                    ──►│
 │                                        │
 │                                        │  Signer processes:
 │                                        │
 │                                        │  1. Parse commitment info:
 │                                        │     received_htlcs = [{H, 100,100, 800,034}]
 │                                        │     (Alice offered → we received)
 │                                        │
 │                                        │  2. Update NodeState.payments:
 │                                        │     payments[H] = RoutedPayment {
 │                                        │       incoming: {CH1: 100,100 sat}
 │                                        │       outgoing: {}
 │                                        │       incoming_cltv_min: 800,034
 │                                        │     }
 │                                        │
 │                                        │  3. Validate payment balance:
 │                                        │     incoming=100,100, outgoing=0
 │                                        │     100,100 + 0 >= 0 ✅ (no outgoing yet)
 │                                        │
 │                                        │  4. Validate commitment structure:
 │                                        │     ✅ fee range OK
 │                                        │     ✅ HTLC count within limit
 │                                        │     ✅ HTLC inflight within limit
 │                                        │     ✅ previous commitment revoked
 │                                        │     ✅ Alice's signature valid
 │                                        │
 │                                        │  5. Update balance delta:
 │                                        │     old_balance = 1,000,000
 │                                        │     new_balance = 899,900 (value to us)
 │                                        │     (100,100 now in HTLC output)
 │                                        │
 │◄── ValidateCommitmentTxReply(        │
 │     next_point=point_2)           ────│
 │                                        │
 │── RevokeCommitmentTx(0) ───────────►│  Revoke our old commitment #0
 │◄── RevokeCommitmentTxReply(         │
 │     secret_0, next_point)         ────│
 │                                        │
 │  ── send revoke_and_ack to Alice ──►  │
```

### 3.2 — We Forward HTLC to Bob (Outgoing)

Now we forward the HTLC to Bob on Channel 2. We subtract our routing fee and reduce the CLTV.

```
Our Node ──update_add_htlc(hash=H, amount=100,000 sat, cltv=800,000)──► Bob
```

Note: amount reduced by 100 sat (routing fee), CLTV reduced by 34 blocks (min delta).

We sign Bob's new commitment (with the HTLC):

```
Node                                    Signer
 │                                        │
 │  Forward HTLC to Bob on Channel 2     │
 │  Sign Bob's commitment #1 with HTLC   │
 │                                        │
 │── SignRemoteCommitmentTx(            │
 │     CH2,                              │
 │     commitment_number=1,              │
 │     feerate=2500,                     │
 │     htlcs=[                           │
 │       { side=local,                   │  "local" = we offered to Bob
 │         amount=100,000 sat,           │  = our OUTGOING
 │         payment_hash=H,              │
 │         cltv_expiry=800,000 }        │
 │     ])                            ──►│
 │                                        │
 │                                        │  Signer processes:
 │                                        │
 │                                        │  1. Parse commitment info:
 │                                        │     This is Bob's commitment where:
 │                                        │     received_htlcs = [{H, 100,000, 800,000}]
 │                                        │     (We offered → Bob receives = our OUTGOING)
 │                                        │
 │                                        │  2. Update NodeState.payments:
 │                                        │     payments[H] = RoutedPayment {
 │                                        │       incoming: {CH1: 100,100 sat}
 │                                        │       outgoing: {CH2: 100,000 sat}
 │                                        │       incoming_cltv_min: 800,034
 │                                        │       outgoing_cltv_max: 800,000
 │                                        │     }
 │                                        │
 │                                        │  3. Validate payment balance:
 │                                        │     incoming = 100,100 sat (from CH1)
 │                                        │     outgoing = 100,000 sat (to CH2)
 │                                        │     100,100 >= 100,000 ✅
 │                                        │     (routing fee = 100 sat retained)
 │                                        │
 │                                        │  4. Validate CLTV delta:
 │                                        │     incoming_cltv = 800,034
 │                                        │     outgoing_cltv = 800,000
 │                                        │     delta = 34 >= min(34) ✅
 │                                        │
 │                                        │  5. Validate commitment structure:
 │                                        │     ✅ All standard commitment checks
 │                                        │
 │◄── SignTxReply(sig) ────────────────│
 │                                        │
 │  ── send commitment_signed to Bob ──►  │
```

### 3.3 — Bob Revokes Old Commitment

```
Node                                    Signer
 │                                        │
 │  ◄── receive revoke_and_ack from Bob   │
 │                                        │
 │── ValidateRevocation(CH2, 0, secret) ►│  Bob revoked commitment #0
 │◄── ValidateRevocationReply ─────────│  ✅ Secret valid
```

### 3.4 — Bob Sends Us New Commitment (Acknowledging Our HTLC)

```
Node                                    Signer
 │                                        │
 │  ◄── commitment_signed from Bob       │
 │  (our commitment #1 on CH2 now        │
 │   has the offered HTLC)               │
 │                                        │
 │── ValidateCommitmentTx(              │
 │     CH2,                              │
 │     commitment_number=1,              │
 │     htlcs=[                           │
 │       { side=local,                   │  We offered to Bob
 │         amount=100,000 sat,           │
 │         payment_hash=H,              │
 │         cltv_expiry=800,000 }        │
 │     ],                                │
 │     signature=bob_sig,                │
 │     htlc_signatures=[bob_htlc_sig]) ►│
 │                                        │
 │                                        │  ✅ Matches what we already signed
 │                                        │     for Bob's commitment
 │                                        │  ✅ Balance checks pass
 │                                        │
 │◄── ValidateCommitmentTxReply ───────│
 │                                        │
 │── RevokeCommitmentTx(CH2, 0) ──────►│  Revoke our old CH2 commitment
 │◄── RevokeCommitmentTxReply ─────────│
 │                                        │
 │  ── send revoke_and_ack to Bob ──►    │
```

### State After Phase 3

```
Signer State:
├── channels:
│   ├── CH1 (Alice): our_balance=899,900, HTLC_incoming=100,100
│   └── CH2 (Bob):   our_balance=900,000, HTLC_outgoing=100,000
├── payments:
│   └── H: { incoming:{CH1: 100,100}, outgoing:{CH2: 100,000},
│            in_cltv: 800,034, out_cltv: 800,000 }
└── excess_amount: 0  (fee not yet earned, HTLC still in-flight)
```

---

## Phase 4: HTLC Settlement (Preimage Flows Back)

Bob receives the preimage from the final recipient (or Bob IS the final recipient). He sends `update_fulfill_htlc` back to us, and we forward the fulfillment to Alice.

### 4.1 — Bob Fulfills HTLC (Sends Preimage)

```
Bob ──update_fulfill_htlc(hash=H, preimage=P)──► Our Node
Bob ──commitment_signed(commitment_2)──► Our Node
```

Bob's new commitment #2 removes the HTLC and gives Bob the 100,000 sat:

```
Node                                    Signer
 │                                        │
 │  Bob fulfilled HTLC H with preimage P │
 │  Bob's new commitment removes HTLC    │
 │  and adds 100,000 to Bob's balance    │
 │                                        │
 │── ValidateCommitmentTx(              │
 │     CH2,                              │
 │     commitment_number=2,              │
 │     feerate=2500,                     │
 │     htlcs=[],                         │  HTLC removed (settled)
 │     to_local=900,000 sat,            │  Our balance stays same
 │     to_remote=100,000 sat,           │  Bob gained HTLC amount
 │     signature=bob_sig)            ──►│
 │                                        │
 │                                        │  Signer processes:
 │                                        │
 │                                        │  1. HTLC removed from CH2:
 │                                        │     payments[H].outgoing = {CH2: 0}
 │                                        │     (outgoing cleared for this channel)
 │                                        │
 │                                        │  2. Balance delta:
 │                                        │     old: 900,000 (to_local in #1)
 │                                        │     new: 900,000 (to_local in #2)
 │                                        │     No change to our direct balance —
 │                                        │     the 100,000 came from the HTLC output
 │                                        │
 │                                        │  3. Validate structure: ✅
 │                                        │
 │◄── ValidateCommitmentTxReply ───────│
 │                                        │
 │── RevokeCommitmentTx(CH2, 1) ──────►│
 │◄── RevokeCommitmentTxReply ─────────│
```

### 4.2 — We Fulfill HTLC to Alice

We now send the preimage back to Alice and remove the incoming HTLC:

```
Our Node ──update_fulfill_htlc(hash=H, preimage=P)──► Alice
```

Sign Alice's new commitment (HTLC removed, our balance increased by 100,100):

```
Node                                    Signer
 │                                        │
 │── SignRemoteCommitmentTx(            │
 │     CH1,                              │
 │     commitment_number=2,              │
 │     htlcs=[],                         │  HTLC removed
 │     to_local=100,100 sat,            │  Alice's balance (she paid)
 │     to_remote=899,900 sat)        ──►│  Our balance (unchanged)
 │                                        │
 │                                        │  Wait — where's the routing fee?
 │                                        │
 │                                        │  Alice paid: 100,100 to HTLC
 │                                        │  Bob received: 100,000
 │                                        │  Difference: 100 sat = routing fee
 │                                        │
 │                                        │  The fee shows up in our balance
 │                                        │  on the NEXT commitment where the
 │                                        │  HTLC is removed:
 │                                        │  old our_balance: 899,900
 │                                        │  new our_balance: 899,900 + ???
 │                                        │
 │                                        │  Actually: the HTLC of 100,100 was
 │                                        │  an output. When removed, 100,100
 │                                        │  is redistributed:
 │                                        │  - 0 to Alice (she paid it)
 │                                        │  - 100,100 to us (we fulfilled it)
 │                                        │  Our new balance: 899,900 + 100,100
 │                                        │                 = 1,000,000
 │                                        │
 │                                        │  But we forwarded 100,000 to Bob.
 │                                        │  Net gain = 100,100 - 100,000 = 100 sat
 │                                        │
 │                                        │  Balance delta tracked:
 │                                        │  excess_amount += 100 sat (fee earned)
 │                                        │
 │◄── SignTxReply(sig) ────────────────│  ✅ All checks pass
 │                                        │
 │  ── send commitment_signed to Alice ►  │
```

### 4.3 — Alice Revokes, We Validate Our New Commitment

```
Node                                    Signer
 │                                        │
 │  ◄── revoke_and_ack from Alice        │
 │── ValidateRevocation(CH1, 0, secret) ►│  ✅
 │                                        │
 │  ◄── commitment_signed from Alice     │
 │  (our CH1 commitment #2, HTLC gone)  │
 │                                        │
 │── ValidateCommitmentTx(              │
 │     CH1,                              │
 │     commitment_number=2,              │
 │     htlcs=[],                         │
 │     to_local=1,000,000 sat,          │  We got the 100,100 back
 │     signature=alice_sig)          ──►│  (we claimed the HTLC)
 │                                        │
 │                                        │  Signer validates:
 │                                        │  ✅ Payment H fully settled
 │                                        │  ✅ Balance delta: +100,100
 │                                        │     (HTLC value returned to us)
 │                                        │  ✅ excess_amount increases by fee
 │                                        │
 │◄── ValidateCommitmentTxReply ───────│
 │                                        │
 │── RevokeCommitmentTx(CH1, 1) ──────►│
 │◄── RevokeCommitmentTxReply ─────────│
```

### State After Phase 4

```
Signer State:
├── channels:
│   ├── CH1 (Alice): our_balance=1,000,000, their_balance=0
│   │                (actually: our=1,000,000 - fee_reserve, since we gained routing fee
│   │                 net: we now have 100 sat more than if no routing happened...
│   │                 Wait: let's recalculate)
│   │
│   │   Actually correct balances after settlement:
│   │   CH1: Alice paid 100,100 total (she had 0 initially - this only works
│   │        if Alice had inbound liquidity. Let me correct the scenario.)
│   │
│   └── [See corrected balance calculation below]
│
├── payments:
│   └── H: settled (preimage known, incoming=outgoing=0)
│
└── excess_amount: 100 sat (routing fee earned!)
```

### Corrected Balance Calculation

Let's be precise. Initially both channels are funded by us:
- CH1: our_balance=1,000,000, alice_balance=0

For Alice to send us an HTLC, Alice needs balance. Let's adjust the scenario:

**Realistic setup:** Alice funded CH1 (she has 1,000,000), we funded CH2 (we have 1,000,000).

| | CH1 (Alice funded) | CH2 (We funded) |
|---|---|---|
| **After open** | alice=1,000,000 us=0 | us=1,000,000 bob=0 |
| **After HTLC added** | alice=899,900 us=0 htlc=100,100 | us=900,000 bob=0 htlc=100,000 |
| **After settlement** | alice=899,900 us=100,100 | us=900,000 bob=100,000 |
| **Net change** | We gained 100,100 | We lost 100,000 |
| **Net profit** | 100,100 - 100,000 = **100 sat routing fee** | |

---

## Phase 5: Accumulate Routing Fees

### Excess Amount Tracking

The signer tracks routing fees via the `excess_amount` accumulator:

```
Before routing:  excess_amount = 0
After 1 route:   excess_amount = 100      (earned 100 sat fee)
After 10 routes: excess_amount = 1,000    (earned 1,000 sat total)
After 100 routes: excess_amount = 10,000  (earned 10,000 sat total)
```

### How Balance Deltas Work

For each commitment update, the signer calculates:

```
balance_delta = (old_balance, new_balance)

excess_amount = excess_amount + new_balance - old_balance
```

The `excess_amount` must never go negative. If it would, the signer rejects:

```
excess_amount + new_balance - old_balance < 0  →  REJECT
(policy-routing-balanced)
```

This means our total balance across all channels can only increase (by routing fees) or stay the same. It can never decrease unless an approved invoice payment is made.

### Multiple Routes

```
Route 1: Alice→Us→Bob    fee=100 sat    excess=100
Route 2: Bob→Us→Alice    fee=150 sat    excess=250
Route 3: Carol→Us→Dave   fee=50 sat     excess=300
...
```

Each successful route adds to the excess. The signer permits these increases because they represent legitimate routing income.

---

## Phase 6: Cooperative Close

After routing for some time, we close channels cooperatively.

### Close Channel 2 (Bob)

```
Node                                    Signer
 │                                        │
 │  Negotiate close with Bob             │
 │  Our balance on CH2: 850,000 sat      │
 │  Bob's balance: 150,000 sat           │
 │                                        │
 │── SignMutualCloseTx(                  │
 │     CH2,                              │
 │     close_tx={                        │
 │       output[0]: us, 849,800 sat      │  (minus close fee)
 │       output[1]: bob, 149,900 sat     │
 │     })                            ──►│
 │                                        │
 │                                        │  Signer validates:
 │                                        │  ✅ policy-mutual-destination-allowlisted
 │                                        │     Our output goes to our wallet
 │                                        │  ✅ policy-mutual-value-matches-commitment
 │                                        │     Values match latest commitment balance
 │                                        │  ✅ policy-mutual-fee-range
 │                                        │     Close fee is reasonable
 │                                        │  ✅ policy-mutual-no-pending-htlcs
 │                                        │     No unresolved HTLCs
 │                                        │
 │◄── SignTxReply(sig) ────────────────│
 │                                        │
 │  ── broadcast close tx ──►            │
 │                                        │
 │── ForgetChannel(bob_id, 2) ─────────►│  Remove from signer state
 │◄── ForgetChannelReply ──────────────│
```

---

## Signer-Side Validation Deep Dive

### The Core Routing Validation (per commitment update)

Every time a commitment is signed or validated, the signer runs this logic:

```
┌─────────────────────────────────────────────────────────┐
│  validate_payments(channel_id, incoming, outgoing,       │
│                    balance_delta, validator)              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  For each PaymentHash in (incoming ∪ outgoing):          │
│                                                          │
│  1. Look up global RoutedPayment for this hash          │
│                                                          │
│  2. Get total incoming across ALL channels:              │
│     total_in = Σ (payments[H].incoming[ch])             │
│     (with this channel's new value substituted)         │
│                                                          │
│  3. Get total outgoing across ALL channels:              │
│     total_out = Σ (payments[H].outgoing[ch])            │
│     (with this channel's new value substituted)         │
│                                                          │
│  4. Get invoice amount if this is our own payment:       │
│     invoiced = invoices[H].amount_msat or None          │
│                                                          │
│  5. validate_payment_balance:                            │
│     ┌────────────────────────────────────────────┐      │
│     │ IF invoiced:                                │      │
│     │   max_to_invoice = invoiced + max_fee       │      │
│     │ ELSE:                                       │      │
│     │   max_to_invoice = 0                        │      │
│     │                                             │      │
│     │ CHECK: total_in + max_to_invoice >= total_out│     │
│     │        OTHERWISE → REJECT                   │      │
│     └────────────────────────────────────────────┘      │
│                                                          │
│  6. validate_payment_cltv (if both bounds known):        │
│     ┌────────────────────────────────────────────┐      │
│     │ CHECK: incoming_cltv > outgoing_cltv        │      │
│     │ CHECK: incoming_cltv - outgoing_cltv >= 34  │      │
│     │        OTHERWISE → REJECT                   │      │
│     └────────────────────────────────────────────┘      │
│                                                          │
│  7. validate balance delta (enforce_balance):            │
│     ┌────────────────────────────────────────────┐      │
│     │ excess_amount += new_balance                 │      │
│     │ excess_amount -= old_balance                 │      │
│     │ CHECK: excess_amount >= 0                    │      │
│     │        OTHERWISE → REJECT                   │      │
│     └────────────────────────────────────────────┘      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### What Gets Checked on Each Message

| Message | Commitment Checks | Routing Checks |
|---------|-------------------|----------------|
| `ValidateCommitmentTx` | Structure, fee, keys, state | Balance, CLTV delta, excess |
| `SignRemoteCommitmentTx` | Structure, fee, keys, state | Balance, CLTV delta, excess |
| `SignMutualCloseTx` | Destination, value, fee | No pending HTLCs |
| `PreapproveInvoice` | N/A | Invoice expiry, velocity |

---

## Routing Fee Economics

### Fee Calculation from Signer's Perspective

```
Incoming HTLC (CH1):  100,100 sat  @ CLTV 800,034
Outgoing HTLC (CH2):  100,000 sat  @ CLTV 800,000

Routing Fee = Incoming - Outgoing = 100 sat
CLTV Delta  = 800,034 - 800,000   = 34 blocks
```

### Signer's Balance Validation Formula

```
For pure routing (no invoices):
  total_incoming_msat >= total_outgoing_msat

For own payments (with invoice):
  total_incoming_msat + invoiced_amount + max_routing_fee >= total_outgoing_msat
```

### Fee Accumulation

The signer doesn't explicitly track "fees earned." Instead, it uses the `excess_amount` accumulator:

- When an HTLC is fulfilled on the incoming side: our balance increases by `incoming_amount`
- When an HTLC is fulfilled on the outgoing side: our balance decreases by `outgoing_amount`
- Net gain (routing fee) is added to `excess_amount`
- If `excess_amount` ever goes negative → policy violation

This elegantly handles complex multi-hop scenarios where HTLCs settle at different times across different channels.

---

## Edge Cases and Error Handling

### Case 1: Incoming HTLC Settles Before Outgoing Is Created

```
Timeline:
  T1: Receive HTLC on CH1 (incoming = 100,100)
  T2: Not yet forwarded (outgoing = 0)

  Validation: 100,100 >= 0 ✅ (always valid — more incoming than outgoing)
```

### Case 2: Outgoing HTLC Created Before Incoming Is Committed

This CANNOT happen legitimately in the protocol — you must receive before you can forward. If the signer sees outgoing without incoming:

```
  incoming = 0, outgoing = 100,000
  Validation: 0 >= 100,000 ❌ → REJECT (policy-commitment-htlc-routing-balance)
```

### Case 3: Multiple Incoming Channels for Same Payment (MPP)

Multi-part payments split across channels:

```
  CH1 incoming: 50,050 sat (hash=H)
  CH3 incoming: 50,050 sat (hash=H)
  CH2 outgoing: 100,000 sat (hash=H)

  total_incoming = 50,050 + 50,050 = 100,100
  total_outgoing = 100,000
  Validation: 100,100 >= 100,000 ✅
```

### Case 4: HTLC Timeout (Payment Fails)

If the HTLC times out without settlement:

```
  HTLC removed from both commitments
  Balance returns to pre-HTLC state
  excess_amount unchanged (no fee earned, no loss either)
  payments[H] cleaned up
```

### Case 5: Rapid Successive Routes

```
  Route 1: H1 incoming=200, outgoing=190  → fee=10,  excess=10
  Route 2: H2 incoming=500, outgoing=480  → fee=20,  excess=30
  Route 3: H3 incoming=1000, outgoing=950 → fee=50,  excess=80

  Each validated independently by payment hash.
  excess_amount grows monotonically for pure routing.
```

### Case 6: CLTV Delta Too Small

```
Node                                    Signer
 │                                        │
 │── SignRemoteCommitmentTx(            │
 │     htlcs=[offered: H,               │
 │            cltv_expiry=800,010])  ──►│
 │                                        │
 │                                        │  payments[H].incoming_cltv = 800,034
 │                                        │  outgoing_cltv = 800,010
 │                                        │  delta = 800,034 - 800,010 = 24
 │                                        │  24 < 34 (minimum) ❌
 │                                        │
 │◄── REJECTED ────────────────────────│  policy-routing-cltv-delta
 │                                        │
 │  ❌ Node must use sufficient delta    │
```

**Why 34 blocks minimum?**
- 3R = 3 × retry timeout blocks (time to claim if first attempt fails)
- 2G = 2 × grace period blocks (network propagation buffer)
- 2S = 2 × settlement blocks (time for tx to confirm)
- Total = 34 blocks ≈ 5.7 hours

If the delta is too small, we risk: outgoing HTLC succeeds (Bob gets paid), but incoming HTLC times out (Alice reclaims) before we can claim the preimage on-chain. Result: we lose the routed amount.

---

## Complete Message Sequence Summary

For one routed payment through two channels, the signer handles:

| # | Message | Channel | Direction | Purpose |
|---|---------|---------|-----------|---------|
| 1 | ValidateCommitmentTx(1) | CH1 | Incoming HTLC | Accept Alice's HTLC |
| 2 | RevokeCommitmentTx(0) | CH1 | - | Revoke old CH1 state |
| 3 | SignRemoteCommitmentTx(1) | CH2 | Outgoing HTLC | Forward HTLC to Bob |
| 4 | ValidateRevocation(0) | CH2 | - | Accept Bob's revocation |
| 5 | ValidateCommitmentTx(1) | CH2 | - | Validate our CH2 commitment |
| 6 | RevokeCommitmentTx(0) | CH2 | - | Revoke old CH2 state |
| 7 | SignRemoteCommitmentTx(1) | CH1 | - | Sign Alice's updated commitment |
| 8 | ValidateRevocation(0) | CH1 | - | Accept Alice's revocation |
| — | *HTLC in flight* | — | — | *Waiting for preimage* |
| 9 | ValidateCommitmentTx(2) | CH2 | HTLC settled | HTLC removed from CH2 |
| 10 | RevokeCommitmentTx(1) | CH2 | - | Revoke CH2 #1 |
| 11 | SignRemoteCommitmentTx(2) | CH1 | HTLC settled | Remove HTLC from Alice |
| 12 | ValidateRevocation(1) | CH1 | - | Accept Alice's revocation |
| 13 | ValidateCommitmentTx(2) | CH1 | - | Validate our CH1 #2 |
| 14 | RevokeCommitmentTx(1) | CH1 | - | Revoke CH1 #1 |

**Total: 14 signer interactions for one routed payment.**

Each of the commitment-related messages (1, 3, 5, 7, 9, 11, 13) triggers the full routing validation: balance check, CLTV check, excess amount tracking, and all structural commitment rules.

---

*Source: [VLS Repository](https://gitlab.com/lightning-signer/validating-lightning-signer) — `vls-core/src/node.rs` (payment tracking), `vls-core/src/policy/simple_validator.rs` (validation logic)*
