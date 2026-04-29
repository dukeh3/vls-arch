# VLS End-to-End Scenarios

This document describes 10 end-to-end scenarios that demonstrate the critical security functions of the Validating Lightning Signer (VLS). Each scenario includes the context, step-by-step message flow between the node and signer, the policy rules involved, and the expected outcome.

These scenarios cover both the happy path and the attack vectors that VLS is specifically designed to defend against.

---

## Table of Contents

1. [Normal Channel Lifecycle](#scenario-1-normal-channel-lifecycle)
2. [Compromised Node Attempts Theft via Fabricated Output](#scenario-2-compromised-node-attempts-theft-via-fabricated-output)
3. [Compromised Node Attempts Fee Siphoning](#scenario-3-compromised-node-attempts-fee-siphoning)
4. [Counterparty Broadcasts Revoked Commitment (Breach/Penalty)](#scenario-4-counterparty-broadcasts-revoked-commitment-breachpenalty)
5. [Compromised Node Tries to Sign a Revoked Commitment](#scenario-5-compromised-node-tries-to-sign-a-revoked-commitment)
6. [HTLC Routing Balance Attack](#scenario-6-htlc-routing-balance-attack)
7. [Compromised Node Redirects Mutual Close Proceeds](#scenario-7-compromised-node-redirects-mutual-close-proceeds)
8. [Transaction Malleation via Non-SegWit Input](#scenario-8-transaction-malleation-via-non-segwit-input)
9. [Compromised Node Attempts Rapid Fund Drainage](#scenario-9-compromised-node-attempts-rapid-fund-drainage)
10. [Force Close and Sweep with Chain Validation](#scenario-10-force-close-and-sweep-with-chain-validation)

---

## Scenario 1: Normal Channel Lifecycle

### Context

The happy-path scenario. Alice's node opens a channel with Bob, routes a payment through it, and cooperatively closes the channel. Every signing request passes validation. This scenario establishes the baseline message flow that all other scenarios deviate from.

### Actors

- **Alice's Node** — Lightning node (CLN or LDK)
- **Alice's Signer** — VLS instance holding Alice's keys
- **Bob** — Counterparty Lightning node

### Preconditions

- Alice's signer has been initialized (`HsmdInit` / `HsmdInit2`)
- Alice has on-chain funds in her wallet
- Bob is a reachable peer

### Step-by-Step Flow

#### Phase 1: Channel Opening

```
Alice's Node                           Alice's Signer
 │                                       │
 │── NewChannel(bob_id, dbid=1) ────────►│  Register new channel
 │◄── NewChannelReply ──────────────────│
 │                                       │
 │── GetChannelBasepoints(bob_id, 1) ──►│  Get our channel keys
 │◄── (basepoints, funding_pubkey) ────│
 │                                       │
 │── GetPerCommitmentPoint(0) ─────────►│  Get first commitment point
 │◄── (point) ─────────────────────────│
 │                                       │
 │  ◄── negotiate open_channel / accept_channel with Bob ──►
 │                                       │
 │── SetupChannel(                  ───►│  Configure channel params
 │     is_outbound=true,                 │
 │     channel_value=1_000_000,          │
 │     push_value=0,                     │
 │     funding_txid, funding_txout,      │
 │     to_self_delay=144,               │
 │     remote_basepoints, ...)           │
 │◄── SetupChannelReply ───────────────│
 │                                       │
 │── SignRemoteCommitmentTx(0) ────────►│  Sign Bob's initial commitment
 │◄── SignTxReply(sig) ────────────────│  ✅ Validates: no HTLCs,
 │                                       │     correct structure,
 │── ValidateCommitmentTx(0, bob_sig) ─►│  Validate our initial commitment
 │◄── ValidateCommitmentTxReply ───────│  ✅ Validates: Bob's signature,
 │                                       │     initial funding value,
 │                                       │     no HTLCs, fee range
```

#### Phase 2: Fund the Channel

```
Alice's Node                           Alice's Signer
 │                                       │
 │── SignWithdrawal(utxos, psbt) ──────►│  Sign funding transaction
 │◄── SignWithdrawalReply(signed_psbt) ─│  ✅ Validates: SegWit inputs,
 │                                       │     known outputs, fee range,
 │  ── broadcast funding tx ──►          │     commitment countersigned,
 │                                       │     no channel push
 │  ... funding confirms (6 blocks) ...  │
 │                                       │
 │── CheckOutpoint(txid, vout) ────────►│  Verify funding is buried
 │◄── CheckOutpointReply(true) ────────│
 │                                       │
 │── LockOutpoint(txid, vout) ─────────►│  Lock the funding UTXO
 │◄── LockOutpointReply ──────────────│
 │                                       │
 │  ◄── channel_ready exchanged with Bob ──►
```

#### Phase 3: Route a Payment (Alice → Carol via Bob)

Alice sends a payment. This requires updating the commitment to add an HTLC.

```
Alice's Node                           Alice's Signer
 │                                       │
 │  Alice wants to send 10,000 sat       │
 │  to Carol, routed through Bob         │
 │                                       │
 │── PreapproveInvoice(carol_invoice) ─►│  Check invoice validity
 │◄── PreapproveInvoiceReply(true) ────│  ✅ Invoice not expired,
 │                                       │     amount within limits
 │                                       │
 │── SignRemoteCommitmentTx(1,          │  Sign Bob's commitment #1
 │     htlcs=[offered: 10,000 sat]) ──►│  (adds our offered HTLC)
 │◄── SignTxReply(sig) ────────────────│  ✅ Validates: HTLC balance,
 │                                       │     fee range, count limits
 │  ── send commitment_signed to Bob ──► │
 │  ◄── receive revoke_and_ack from Bob  │
 │                                       │
 │── ValidateRevocation(0, secret) ────►│  Accept Bob's revocation of #0
 │◄── ValidateRevocationReply ─────────│  ✅ Secret matches expected
 │                                       │
 │  ◄── receive commitment_signed(1)     │
 │                                       │
 │── ValidateCommitmentTx(1, bob_sig,   │  Validate our commitment #1
 │     htlcs=[offered: 10,000 sat]) ──►│
 │◄── ValidateCommitmentTxReply ───────│  ✅ Validates all rules
 │                                       │
 │── RevokeCommitmentTx(0) ───────────►│  Revoke our commitment #0
 │◄── RevokeCommitmentTxReply(secret) ─│
 │                                       │
 │  ── send revoke_and_ack to Bob ─────► │
 │                                       │
 │  ... HTLC settles (preimage flows     │
 │      back from Carol → Bob → Alice)   │
 │  ... commitments #2 exchanged to      │
 │      remove settled HTLC ...          │
```

#### Phase 4: Cooperative Close

```
Alice's Node                           Alice's Signer
 │                                       │
 │  ◄── negotiate closing_signed ──►     │
 │                                       │
 │── SignMutualCloseTx(close_tx) ──────►│  Sign close transaction
 │◄── SignTxReply(sig) ────────────────│  ✅ Validates: destination
 │                                       │     allowlisted, value matches
 │  ── broadcast mutual close tx ──►     │     commitment, fee in range,
 │                                       │     no pending HTLCs
 │── ForgetChannel(bob_id, 1) ─────────►│  Clean up state
 │◄── ForgetChannelReply ──────────────│
```

### Policy Rules Exercised

| Rule | When |
|------|------|
| `policy-channel-contest-delay-range-*` | SetupChannel |
| `policy-channel-safe-mode` | SetupChannel |
| `policy-commitment-first-no-htlcs` | Commitment #0 |
| `policy-commitment-initial-funding-value` | Commitment #0 |
| `policy-onchain-funding-non-malleable` | SignWithdrawal |
| `policy-onchain-no-unknown-outputs` | SignWithdrawal |
| `policy-onchain-fee-range` | SignWithdrawal |
| `policy-onchain-initial-commitment-countersigned` | SignWithdrawal |
| `policy-commitment-fee-range` | All commitments |
| `policy-commitment-htlc-routing-balance` | Commitment #1 |
| `policy-commitment-previous-revoked` | Commitment #1+ |
| `policy-invoice-not-expired` | PreapproveInvoice |
| `policy-mutual-destination-allowlisted` | SignMutualCloseTx |
| `policy-mutual-value-matches-commitment` | SignMutualCloseTx |
| `policy-mutual-fee-range` | SignMutualCloseTx |
| `policy-mutual-no-pending-htlcs` | SignMutualCloseTx |

### Expected Outcome

All signing requests succeed. The channel opens, a payment is routed, and the channel closes cooperatively. Both parties recover their correct balances on-chain.

---

## Scenario 2: Compromised Node Attempts Theft via Fabricated Output

### Context

An attacker has gained control of Alice's Lightning node but not the VLS signer. The attacker attempts to steal funds by adding an extra output to the funding transaction that sends coins to an address the attacker controls.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer (separate process/device)
- **Goal:** Attacker wants to siphon wallet funds during a channel open

### Step-by-Step Flow

```
Compromised Node                       Alice's Signer
 │                                       │
 │  ... channel setup completes ...      │
 │                                       │
 │── SignWithdrawal(                     │
 │     utxos=[wallet UTXO: 2 BTC],      │
 │     psbt={                            │
 │       output[0]: channel funding      │
 │                  1,000,000 sat        │
 │       output[1]: change to wallet     │  Attacker constructs PSBT
 │                  900,000 sat          │  with a theft output
 │       output[2]: ATTACKER ADDRESS  ◄──│
 │                  100,000 sat          │
 │     })                            ──►│
 │                                       │
 │                                       │  Signer analyzes all outputs:
 │                                       │  - output[0]: ✅ funds channel
 │                                       │  - output[1]: ✅ wallet change
 │                                       │  - output[2]: ❌ UNKNOWN
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-onchain-no-unknown-outputs
 │                                       │
 │  ❌ Attacker cannot steal funds       │
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-onchain-no-unknown-outputs`** | Primary defense — every output must be wallet, allowlisted, or channel funding |
| `policy-onchain-fee-range` | Secondary defense — if the "theft" is disguised as fee overpayment |

### Why This Matters

Without VLS, a compromised node has full signing authority and could construct any transaction it wants. With VLS, even though the attacker controls the node, they cannot get a signature on a transaction that sends funds anywhere the signer hasn't approved. The signer independently verifies that every satoshi is accounted for.

### Expected Outcome

**Signing refused.** The signer returns a policy violation error. No funds leave Alice's wallet. The attacker's theft attempt is blocked without any human intervention.

---

## Scenario 3: Compromised Node Attempts Fee Siphoning

### Context

A more subtle attack than adding an explicit theft output. The attacker sets an absurdly high fee on a commitment transaction or on-chain transaction. The excess fee effectively goes to miners — and if the attacker is colluding with a miner (or running one), they recover the overpaid fees.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer
- **Goal:** Drain channel balance through excessive fees, recoverable by colluding miner

### Step-by-Step Flow

#### Attack Vector A: Commitment Fee Manipulation

```
Compromised Node                       Alice's Signer
 │                                       │
 │  Channel: 1,000,000 sat capacity      │
 │  Alice's balance: 800,000 sat         │
 │  Bob's balance: 200,000 sat           │
 │                                       │
 │  Attacker agrees to Bob's proposal    │
 │  for a commitment with feerate        │
 │  of 500,000 sat/kw (absurdly high)    │
 │                                       │
 │── ValidateCommitmentTx(n+1,           │
 │     feerate=500,000,                  │
 │     to_local=50,000,                  │  750,000 sat "fee" =
 │     to_remote=200,000,               │  effectively stolen
 │     sig=bob_sig)                  ──►│
 │                                       │
 │                                       │  Signer checks feerate:
 │                                       │  500,000 sat/kw >> 25,000 max
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-commitment-fee-range
 │                                       │
 │  ❌ Cannot drain via fees             │
```

#### Attack Vector B: On-chain Fee Manipulation

```
Compromised Node                       Alice's Signer
 │                                       │
 │── SignWithdrawal(                     │
 │     utxos=[2 BTC wallet UTXO],       │
 │     psbt={                            │
 │       output[0]: channel 1,000,000    │
 │       output[1]: change 10,000     ◄──│  990,000 sat "fee"
 │     })                            ──►│
 │                                       │
 │                                       │  Signer calculates fee:
 │                                       │  200,000,000 - 1,000,000 -
 │                                       │  10,000 = 198,990,000 sat
 │                                       │  feerate = way beyond max
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-onchain-fee-range
 │                                       │
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-commitment-fee-range`** | Blocks excessive commitment tx fees (default max: 25,000 sat/kw) |
| **`policy-onchain-fee-range`** | Blocks excessive on-chain tx fees |
| `policy-mutual-fee-range` | Blocks excessive cooperative close fees |
| `policy-htlc-fee-range` | Blocks excessive HTLC transaction fees |

### Why This Matters

Fee siphoning is a subtle attack because fees are a normal and expected part of every transaction. The attacker doesn't need to add suspicious outputs — they just inflate the fee. Without VLS's fee-range validation, this is invisible at the protocol level. VLS enforces reasonable fee bounds on every transaction type.

### Expected Outcome

**Signing refused** for any transaction with fees outside the configured range. The signer protects against fee manipulation across all transaction types: commitments, on-chain, mutual close, and HTLC transactions.

---

## Scenario 4: Counterparty Broadcasts Revoked Commitment (Breach/Penalty)

### Context

Bob attempts to cheat by broadcasting an old (revoked) commitment transaction where his balance was higher. Alice's node detects the breach and asks VLS to sign a penalty (justice) transaction that claims all channel funds.

This is the scenario VLS is designed to support — the signer must correctly sign the penalty transaction using the revocation secret that Bob disclosed when he revoked that state.

### Threat Model

- **Attacker:** Bob (counterparty)
- **Honest:** Alice's node and signer
- **Goal (attacker):** Steal funds by publishing favorable old state
- **Goal (defender):** Claim all funds via penalty transaction

### Preconditions

- Channel has gone through multiple commitment updates
- At state #5: Alice=800,000 sat, Bob=200,000 sat
- Bob broadcasts revoked commitment #2 where Bob=500,000 sat

### Step-by-Step Flow

```
Bob                     Alice's Node                    Alice's Signer
 │                        │                                │
 │  broadcasts revoked    │                                │
 │  commitment #2 ──────►│  Detects old commitment        │
 │  (Bob=500k, Alice=500k)│  on-chain. Looks up           │
 │                        │  revocation secret for #2      │
 │                        │                                │
 │                        │── SignPenaltyToUs(             │
 │                        │     revocation_secret=#2,      │
 │                        │     tx=penalty_tx,             │
 │                        │     wscript=to_local_script)──►│
 │                        │                                │
 │                        │                                │  Signer validates:
 │                        │                                │  ✅ Secret is valid for
 │                        │                                │     commitment #2
 │                        │                                │  ✅ Penalty tx sweeps
 │                        │                                │     to wallet/allowlist
 │                        │                                │  ✅ Locktime reasonable
 │                        │                                │  ✅ Sequence valid
 │                        │                                │
 │                        │◄── SignTxReply(sig) ───────────│
 │                        │                                │
 │                        │  broadcast penalty tx ─────────►
 │                        │  Claims ALL 1,000,000 sat      │
 │                        │                                │
 │  Bob loses everything  │                                │
```

### Policy Rules Exercised

| Rule | Role |
|------|------|
| `policy-sweep-destination-allowlisted` | Penalty output goes to our wallet |
| `policy-sweep-locktime` | Locktime is not unreasonably far in future |
| `policy-sweep-sequence` | Correct sequence for spending revoked output |
| `policy-sweep-version` | Transaction version is 2 |

### Supporting Infrastructure

This scenario depends on the full state management chain working correctly throughout the channel lifetime:

| Rule | When | Purpose |
|------|------|---------|
| `policy-commitment-previous-revoked` | Every commitment update | Ensures we collect revocation secrets |
| `ValidateRevocation` | Every revoke_and_ack | Stores Bob's revocation secrets |

### Why This Matters

The penalty mechanism is Lightning's core security model. VLS ensures that:
1. Revocation secrets are properly collected and validated at every state transition
2. When a breach occurs, the penalty transaction is correctly signed
3. Penalty proceeds go to the rightful owner (allowlisted destination)

Without VLS, if the node itself was compromised, the attacker could prevent the penalty transaction from being signed or redirect its output.

### Expected Outcome

**Penalty transaction signed and broadcast.** Alice claims the entire channel balance (1,000,000 sat). Bob's cheating attempt costs him everything.

---

## Scenario 5: Compromised Node Tries to Sign a Revoked Commitment

### Context

An attacker who has compromised Alice's node attempts a particularly dangerous attack: tricking VLS into signing Alice's own revoked commitment. If broadcast, the counterparty (Bob) would detect the breach and claim all funds via penalty.

This is the inverse of Scenario 4 — instead of the counterparty cheating, the attacker tries to make Alice cheat, knowing that Bob will punish her.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer
- **Colluding:** Bob (counterparty) — waiting to claim penalty
- **Goal:** Alice broadcasts revoked state → Bob claims all funds → Attacker and Bob split

### Step-by-Step Flow

```
Compromised Node                       Alice's Signer
 │                                       │
 │  Channel state history:               │
 │  #0: Alice=1,000,000 Bob=0           │
 │  #1: Alice=900,000  Bob=100,000      │
 │  #2: Alice=800,000  Bob=200,000      │
 │  #3: Alice=700,000  Bob=300,000 ◄── current
 │                                       │
 │  Attacker wants to sign commitment    │
 │  #0 (Alice=1,000,000) and broadcast   │
 │  it. But the real goal is to trigger  │
 │  Bob's penalty (colluding with Bob).  │
 │                                       │
 │  Attempt 1: Try to get signature on   │
 │  old commitment #0                    │
 │                                       │
 │── SignLocalCommitmentTx2(             │
 │     commitment_number=0)          ──►│
 │                                       │
 │                                       │  Signer checks:
 │                                       │  Current state = #3
 │                                       │  Commitment #0 was revoked
 │                                       │  at state transition #0→#1
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-commitment-holder-
 │                                       │  not-revoked
 │                                       │
 │  Attempt 2: Try commitment #1         │
 │── SignLocalCommitmentTx2(             │
 │     commitment_number=1)          ──►│
 │◄── REJECTED ────────────────────────│  Same: already revoked
 │                                       │
 │  Attempt 3: Try commitment #2         │
 │── SignLocalCommitmentTx2(             │
 │     commitment_number=2)          ──►│
 │◄── REJECTED ────────────────────────│  Same: already revoked
 │                                       │
 │  ❌ Cannot sign any revoked state     │
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-commitment-holder-not-revoked`** | Primary defense — refuses to sign any commitment whose revocation secret has been disclosed |
| `policy-commitment-retry-same` | Prevents re-signing with different data at the same commitment number |

### Why This Matters

This is one of the most important protections VLS provides. In a hot-wallet setup, the attacker could simply extract the private key and sign anything. With VLS:

- The private key never leaves the signer
- The signer tracks which commitments have been revoked
- It will never produce a signature on a revoked state

Even if the attacker has full control of the node, they cannot trigger the penalty mechanism against Alice. The signer's state tracking makes this attack impossible.

### Expected Outcome

**All signing attempts refused.** The signer will only sign the current (non-revoked) commitment. Alice's funds remain safe. The collusion between the attacker and Bob yields nothing.

---

## Scenario 6: HTLC Routing Balance Attack

### Context

Alice runs a routing node. An attacker compromises it and tries to route a payment where the outgoing HTLC is not backed by an incoming HTLC of equal or greater value. This would effectively transfer Alice's funds to the outgoing channel's counterparty without receiving anything in return.

### Threat Model

- **Compromised:** Alice's routing node
- **Secure:** Alice's VLS signer
- **Goal:** Send funds out via HTLC without corresponding incoming funds

### Preconditions

- Alice has two channels: Alice↔Bob (500,000 sat) and Alice↔Carol (500,000 sat)
- Attacker wants to drain Alice→Carol channel by offering an HTLC without a matching inbound

### Step-by-Step Flow

```
Compromised Node                       Alice's Signer
 │                                       │
 │  Attacker creates a fake payment:     │
 │  Offer 400,000 sat HTLC to Carol     │
 │  with NO incoming HTLC from anyone    │
 │                                       │
 │── SignRemoteCommitmentTx(             │
 │     Alice↔Carol channel,              │
 │     commitment_number=n+1,            │
 │     htlcs=[                           │
 │       offered: 400,000 sat  ◄─────────│  No matching incoming HTLC
 │     ],                                │
 │     feerate=1000)                 ──►│
 │                                       │
 │                                       │  Signer checks HTLC balance:
 │                                       │
 │                                       │  Offered HTLCs:  400,000 sat
 │                                       │  Received HTLCs: 0 sat
 │                                       │  Invoiced:       0 sat
 │                                       │
 │                                       │  400,000 > 0 → IMBALANCED
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-commitment-htlc-
 │                                       │  routing-balance
 │                                       │
 │  ❌ Cannot send unbalanced HTLCs      │
```

### What About Legitimate Payments?

For Alice's own (non-routed) payments, the signer uses invoice preapproval:

```
Legitimate Node                        Alice's Signer
 │                                       │
 │── PreapproveInvoice(carol_invoice) ─►│  Invoice for 50,000 sat
 │◄── PreapproveInvoiceReply(true) ────│  ✅ Valid, not expired,
 │                                       │     within velocity limit
 │                                       │
 │── SignRemoteCommitmentTx(             │
 │     htlcs=[offered: 50,000 sat]) ──►│
 │                                       │  Signer checks: offered HTLC
 │                                       │  is covered by approved invoice
 │◄── SignTxReply(sig) ────────────────│  ✅ Signed
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-commitment-htlc-routing-balance`** | Offered amount must be covered by received HTLCs or approved invoices |
| `policy-commitment-payment-invoiced` | For own payments, amount must not exceed invoice |
| `policy-commitment-payment-velocity` | Rate-limits total outgoing payments |
| `policy-routing-cltv-delta` | Ensures safe timing between incoming/outgoing HTLCs |

### Why This Matters

Routing is a core Lightning function, and the balance between incoming and outgoing HTLCs is a fundamental safety invariant. A routing node should never pay out more than it receives. Without this check, a compromised routing node could drain all channels by generating one-sided HTLCs.

### Expected Outcome

**Signing refused.** The signer will not sign a commitment where offered HTLCs exceed received HTLCs plus approved invoices. The attacker cannot drain routing channels.

---

## Scenario 7: Compromised Node Redirects Mutual Close Proceeds

### Context

During a cooperative channel close, the node normally specifies where its share of funds should go (the "shutdown script"). An attacker who controls the node attempts to substitute this with their own address, redirecting Alice's funds on close.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer
- **Goal:** Redirect cooperative close output to attacker's address

### Preconditions

- Channel with Bob, Alice's balance: 800,000 sat
- Alice specified `upfront_shutdown_script` at channel open (optional, strengthens defense)

### Step-by-Step Flow

#### Attack Vector A: Upfront Shutdown Script Was Set

```
Compromised Node                       Alice's Signer
 │                                       │
 │  At channel open, Alice set:          │
 │  upfront_shutdown_script =            │
 │    bc1q_alice_wallet_address          │
 │                                       │
 │  Attacker negotiates close with Bob,  │
 │  substituting the shutdown script:    │
 │                                       │
 │── SignMutualCloseTx(                  │
 │     close_tx={                        │
 │       output[0]: bc1q_ATTACKER ◄──────│  Attacker's address
 │                  800,000 sat          │
 │       output[1]: Bob                  │
 │                  200,000 sat          │
 │     })                            ──►│
 │                                       │
 │                                       │  Signer checks:
 │                                       │  upfront_shutdown_script =
 │                                       │    bc1q_alice_wallet_address
 │                                       │  output[0] =
 │                                       │    bc1q_ATTACKER
 │                                       │  MISMATCH ❌
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-mutual-destination-
 │                                       │  allowlisted
```

#### Attack Vector B: No Upfront Script, Attacker Uses Non-Wallet Address

```
Compromised Node                       Alice's Signer
 │                                       │
 │── SignMutualCloseTx(                  │
 │     close_tx={                        │
 │       output[0]: bc1q_ATTACKER        │
 │                  800,000 sat          │
 │     })                            ──►│
 │                                       │
 │                                       │  Signer checks:
 │                                       │  - Is bc1q_ATTACKER in wallet?
 │                                       │    NO ❌
 │                                       │  - Is it in allowlist?
 │                                       │    NO ❌
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-mutual-destination-
 │                                       │  allowlisted
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-mutual-destination-allowlisted`** | Close output must go to wallet or allowlisted address |
| `policy-mutual-value-matches-commitment` | Our output value must match our balance |
| `policy-mutual-fee-range` | Can't hide theft as excessive fee |
| `policy-mutual-no-pending-htlcs` | Can't close with HTLCs pending (prevents partial loss) |

### Why This Matters

Cooperative close is the most common way channels end. It's a single on-chain transaction that distributes the final balances. If the close destination can be substituted, it's a complete theft of the channel balance. VLS ensures the proceeds go to a destination the signer can verify belongs to Alice.

### Expected Outcome

**Signing refused.** The signer only signs close transactions where Alice's output goes to her wallet or an explicitly allowlisted address. The attacker cannot redirect close proceeds.

---

## Scenario 8: Transaction Malleation via Non-SegWit Input

### Context

The attacker constructs a funding transaction that uses a legacy (non-SegWit) input. A third party (or the attacker themselves) malleates the transaction by modifying the signature encoding, which changes the txid. Since all commitment transactions reference the funding txid, malleation invalidates every commitment, permanently locking the channel funds in the 2-of-2 multisig.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer
- **Goal:** Lock funds permanently by malleating the funding transaction

### Background: Why Malleation Is Dangerous

```
Before malleation:
  Funding TX (txid: abc123) → Commitment TX spends abc123:0 ✅

After malleation:
  Funding TX (txid: def456) → Commitment TX still spends abc123:0 ❌
                               abc123:0 doesn't exist, commitments are invalid
                               Funds in def456:0 are permanently locked
```

### Step-by-Step Flow

```
Compromised Node                       Alice's Signer
 │                                       │
 │  Attacker constructs funding tx       │
 │  using a legacy P2PKH input:          │
 │                                       │
 │── SignWithdrawal(                     │
 │     utxos=[                           │
 │       { txid: ...,                    │
 │         script: P2PKH ◄───────────────│  Non-SegWit input
 │       }                               │
 │     ],                                │
 │     psbt=funding_psbt)            ──►│
 │                                       │
 │                                       │  Signer inspects inputs:
 │                                       │  input[0] script type = P2PKH
 │                                       │  P2PKH is NOT SegWit
 │                                       │  → Transaction is malleable
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-onchain-funding-
 │                                       │  non-malleable
 │                                       │
 │  ❌ Cannot create malleable funding   │
```

### What the Signer Accepts

```
Legitimate Node                        Alice's Signer
 │                                       │
 │── SignWithdrawal(                     │
 │     utxos=[                           │
 │       { txid: ...,                    │
 │         script: P2WPKH ◄──────────────│  Native SegWit ✅
 │       }                               │
 │     ],                                │
 │     psbt=funding_psbt)            ──►│
 │                                       │
 │◄── SignWithdrawalReply(signed_psbt) ─│  ✅ SegWit inputs only
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-onchain-funding-non-malleable`** | Requires all funding tx inputs to be SegWit |
| `policy-onchain-format-standard` | Ensures standard transaction format |

### Why This Matters

Transaction malleation was one of the original motivations for SegWit (BIP 141). Lightning absolutely depends on txid stability — every commitment transaction is pre-signed referencing the funding txid. If the txid changes after broadcast, all pre-signed transactions become invalid. VLS enforces SegWit-only inputs at the signing layer, making this attack structurally impossible.

### Expected Outcome

**Signing refused.** The signer only signs funding transactions with SegWit (non-malleable) inputs. Channel funds can never be locked by malleation.

---

## Scenario 9: Compromised Node Attempts Rapid Fund Drainage

### Context

An attacker who has compromised Alice's node tries to drain funds as quickly as possible before the compromise is detected. They create invoices on colluding nodes and attempt to pay them all rapidly, emptying every channel.

### Threat Model

- **Compromised:** Alice's Lightning node
- **Secure:** Alice's VLS signer
- **Goal:** Drain all channel balances to attacker-controlled nodes before detection

### Preconditions

- Alice has 5 channels, 1 BTC total capacity
- Attacker has colluding nodes ready to receive payments

### Step-by-Step Flow

```
Compromised Node                       Alice's Signer
 │                                       │
 │  ── Round 1: Pay 200,000 sat ──       │
 │                                       │
 │── PreapproveInvoice(invoice_1) ─────►│  Check invoice
 │◄── PreapproveInvoiceReply(true) ────│  ✅ First payment, within limits
 │                                       │
 │── SignRemoteCommitmentTx(htlc:200k) ►│  Sign commitment with HTLC
 │◄── SignTxReply(sig) ────────────────│  ✅ Covered by invoice
 │                                       │
 │  ── Round 2: Pay 200,000 sat ──       │
 │                                       │
 │── PreapproveInvoice(invoice_2) ─────►│  Check invoice
 │◄── PreapproveInvoiceReply(true) ────│  ✅ Still within limits
 │                                       │
 │  ── Round 3: Pay 200,000 sat ──       │
 │                                       │
 │── PreapproveInvoice(invoice_3) ─────►│  Check invoice
 │                                       │  Signer checks velocity:
 │                                       │  600,000 sat in short period
 │                                       │  EXCEEDS velocity limit
 │                                       │
 │◄── PreapproveInvoiceReply(false) ───│  policy-commitment-payment-
 │                                       │  velocity (WARNING/REJECT)
 │                                       │
 │  ── Round 3 alt: Skip preapproval ──  │
 │                                       │
 │── SignRemoteCommitmentTx(             │
 │     htlc: 200,000 sat,               │
 │     no matching invoice)          ──►│
 │                                       │
 │                                       │  No approved invoice for this
 │                                       │  HTLC and no incoming HTLC
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-commitment-htlc-
 │                                       │  routing-balance +
 │                                       │  policy-commitment-payment-
 │                                       │  invoiced
 │                                       │
 │  ── Round 3 alt 2: Overpay invoice ── │
 │                                       │
 │── PreapproveInvoice(                  │
 │     invoice for 50,000 sat)       ──►│
 │◄── PreapproveInvoiceReply(true) ────│  ✅ Small amount
 │                                       │
 │── SignRemoteCommitmentTx(             │
 │     htlc: 500,000 sat ◄──────────────│  10x the invoice amount
 │     )                             ──►│
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-commitment-payment-
 │                                       │  invoiced (amount exceeds
 │                                       │  invoice)
```

### Policy Rules That Block This Attack

| Rule | Role |
|------|------|
| **`policy-commitment-payment-velocity`** | Rate-limits total outgoing value per time window |
| **`policy-commitment-payment-invoiced`** | Payment amount must not exceed invoice amount |
| `policy-commitment-htlc-routing-balance` | Cannot send without matching incoming or approved invoice |
| `policy-commitment-htlc-inflight-limit` | Caps total in-flight HTLC value |
| `policy-invoice-not-expired` | Rejects expired invoices |

### Defense in Depth

Even if one rule is bypassed, multiple layers protect against drainage:

| Layer | Limit | Effect |
|-------|-------|--------|
| Per-invoice validation | Exact amount match | Cannot overpay |
| Velocity limit | X sat per time window | Caps drainage rate |
| HTLC inflight limit | Max concurrent HTLC value | Caps instantaneous exposure |
| Routing balance | Must have matching inbound | Cannot create one-sided outflows |

### Why This Matters

This is the most realistic attack scenario — a compromised node simply trying to send all funds to the attacker. VLS provides multiple independent rate-limiting and validation mechanisms that work together to minimize damage even in a sustained attack. The velocity limit is particularly important because it caps how much can be stolen before the compromise is detected and remediated.

### Expected Outcome

**Drainage is rate-limited and eventually blocked.** The attacker may succeed in sending some payments within the velocity window, but cannot drain all funds. The velocity limit buys time for the operator to detect the compromise and shut down the node. Multiple independent checks ensure no single bypass drains everything.

---

## Scenario 10: Force Close and Sweep with Chain Validation

### Context

Alice's node loses connectivity with Bob (or Bob becomes unresponsive). Alice must force-close the channel by broadcasting her latest commitment transaction. After the CSV delay expires, she needs VLS to sign a sweep transaction to recover her funds from the to_local output. The signer validates every aspect of the sweep to ensure funds return to Alice.

### Threat Model

- **Both honest** (or potentially compromised node attempting to redirect sweep)
- **Secure:** Alice's VLS signer
- **Goal:** Safely recover funds after force-close

### Preconditions

- Channel with Bob, Alice's balance: 700,000 sat
- Negotiated `to_self_delay`: 144 blocks
- Alice's latest commitment is #5

### Step-by-Step Flow

#### Phase 1: Force Close — Broadcast Commitment

```
Alice's Node                           Alice's Signer
 │                                       │
 │  Bob is unresponsive. Initiate        │
 │  force close.                         │
 │                                       │
 │── SignLocalCommitmentTx2(             │
 │     commitment_number=5)          ──►│
 │                                       │
 │                                       │  Signer checks:
 │                                       │  ✅ Commitment #5 is current
 │                                       │  ✅ Not revoked
 │                                       │
 │◄── SignCommitmentTxReply(sig) ──────│
 │                                       │
 │  Combine Alice's sig + Bob's sig      │
 │  (from ValidateCommitmentTx #5)       │
 │  ── broadcast commitment #5 ────────► │
```

#### Phase 2: Wait for CSV Delay

```
 │  ... 144 blocks pass ...              │
 │                                       │
 │  Commitment #5 outputs:               │
 │  - to_local: 700,000 sat (ours,       │
 │    locked by CSV 144 blocks)          │
 │  - to_remote: 300,000 sat (Bob's,     │
 │    immediately spendable by Bob)      │
```

#### Phase 3: Sweep to_local Output

```
Alice's Node                           Alice's Signer
 │                                       │
 │── SignDelayedPaymentToUs(             │
 │     commitment_number=5,              │
 │     tx=sweep_tx{                      │
 │       input: commitment_#5:to_local   │
 │       output: bc1q_alice_wallet       │
 │       locktime: current_height        │
 │       sequence: 144 (CSV delay)       │
 │     },                                │
 │     wscript=to_local_script)      ──►│
 │                                       │
 │                                       │  Signer validates:
 │                                       │
 │                                       │  ✅ policy-sweep-version
 │                                       │     tx version = 2
 │                                       │
 │                                       │  ✅ policy-sweep-sequence
 │                                       │     sequence = 144 (matches
 │                                       │     negotiated CSV delay)
 │                                       │
 │                                       │  ✅ policy-sweep-locktime
 │                                       │     locktime ≤ current_height
 │                                       │     + MAX_CHAIN_LAG (2 blocks)
 │                                       │
 │                                       │  ✅ policy-sweep-destination-
 │                                       │     allowlisted
 │                                       │     bc1q_alice_wallet is in
 │                                       │     wallet derivation path
 │                                       │
 │◄── SignTxReply(sig) ────────────────│
 │                                       │
 │  ── broadcast sweep tx ─────────────► │
 │  Alice recovers 700,000 sat           │
 │  (minus sweep tx fee)                 │
```

#### What If the Node Tries to Redirect the Sweep?

```
Compromised Node                       Alice's Signer
 │                                       │
 │── SignDelayedPaymentToUs(             │
 │     tx=sweep_tx{                      │
 │       output: bc1q_ATTACKER ◄─────────│  Attacker's address
 │     })                            ──►│
 │                                       │
 │                                       │  ✅ Version OK
 │                                       │  ✅ Sequence OK
 │                                       │  ❌ Destination NOT in wallet
 │                                       │     and NOT allowlisted
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-sweep-destination-
 │                                       │  allowlisted
```

#### What If the Node Uses a Bad Locktime?

```
Compromised Node                       Alice's Signer
 │                                       │
 │── SignDelayedPaymentToUs(             │
 │     tx=sweep_tx{                      │
 │       locktime: current + 1000 ◄──────│  Far future locktime
 │     })                            ──►│
 │                                       │
 │                                       │  current_height = 850,000
 │                                       │  locktime = 851,000
 │                                       │  851,000 > 850,002 (max)
 │                                       │
 │◄── REJECTED ────────────────────────│  policy-sweep-locktime
```

### Policy Rules Exercised

| Rule | Role |
|------|------|
| `policy-commitment-holder-not-revoked` | Only sign current commitment for broadcast |
| **`policy-sweep-version`** | Transaction version must be 2 |
| **`policy-sweep-sequence`** | Sequence matches CSV delay |
| **`policy-sweep-locktime`** | Locktime not too far in future |
| **`policy-sweep-destination-allowlisted`** | Output goes to our wallet |
| `policy-chain-validated` | Chain state is properly attested |

### Chain Synchronization Role

The signer uses blockchain data (via `AddBlock` messages) to validate time-sensitive operations:

```
Node                                   Alice's Signer
 │                                       │
 │── AddBlock(header, txoo_proof) ─────►│  Feed new block
 │◄── AddBlockReply ───────────────────│
 │                                       │  Signer now knows current
 │                                       │  height, can validate:
 │                                       │  - Sweep locktime is current
 │                                       │  - HTLC timelocks are valid
 │                                       │  - Funding is confirmed
```

### Why This Matters

Force-close is the last-resort fund recovery mechanism in Lightning. Every aspect must work correctly:
- The right commitment (current, non-revoked) must be broadcast
- The sweep must go to the right address
- The timing must be correct
- The signer must independently verify all of this

Even in a force-close scenario with a compromised node, VLS ensures the sweep transaction sends funds back to Alice's wallet and nowhere else.

### Expected Outcome

**Force-close and sweep succeed.** The commitment is broadcast, the CSV delay passes, and the sweep transaction returns Alice's 700,000 sat (minus fees) to her wallet. All destination, timing, and format validations pass. If the node is compromised and tries to redirect the sweep, the signer refuses.

---

## Summary Matrix

| # | Scenario | Attack Vector | Primary Defense | Outcome |
|---|----------|---------------|-----------------|---------|
| 1 | Normal lifecycle | None (happy path) | All rules | All operations succeed |
| 2 | Fabricated output | Extra theft output in tx | `policy-onchain-no-unknown-outputs` | Signing refused |
| 3 | Fee siphoning | Excessive fees | `policy-*-fee-range` | Signing refused |
| 4 | Breach/penalty | Counterparty publishes old state | Revocation secret + sweep rules | Penalty tx signed, all funds claimed |
| 5 | Sign revoked commitment | Trick signer into signing old state | `policy-commitment-holder-not-revoked` | Signing refused |
| 6 | HTLC routing imbalance | Unbalanced outgoing HTLCs | `policy-commitment-htlc-routing-balance` | Signing refused |
| 7 | Redirect close proceeds | Substitute shutdown script | `policy-mutual-destination-allowlisted` | Signing refused |
| 8 | Tx malleation | Non-SegWit funding input | `policy-onchain-funding-non-malleable` | Signing refused |
| 9 | Rapid fund drainage | Mass payments to attacker | `policy-*-velocity` + `policy-*-invoiced` | Rate-limited, then blocked |
| 10 | Force close sweep | Redirect sweep output | `policy-sweep-destination-allowlisted` | Sweep to wallet only |

---

*Source: [VLS Repository](https://gitlab.com/lightning-signer/validating-lightning-signer) — Policy implementation in `vls-core/src/policy/`, message handling in `vls-protocol-signer/src/handler.rs`*
