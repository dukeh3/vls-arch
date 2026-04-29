# VLS Policy Rules — Complete Reference

This document provides a detailed explanation of all **92 policy validation rules** enforced by the [Validating Lightning Signer (VLS)](https://gitlab.com/lightning-signer/validating-lightning-signer). These rules are the core security mechanism that protects Lightning funds even when the Lightning node is fully compromised.

## How Policy Enforcement Works

VLS validates every signing request against these rules before producing a signature. The validation pipeline works in two phases:

1. **Phase 1 — Recomposition**: The signer reconstructs the expected transaction from its own state and compares sighashes. This catches structural and format violations implicitly.
2. **Phase 2 — Policy Validation**: Explicit checks against business rules, economic limits, and protocol correctness.

Rules are categorized as:
- **Mandatory** — Always enforced; violations result in signing refusal
- **Optional** — Use-case specific; can be enabled per deployment (e.g., merchant mode, routing hub mode)

Implementation source: `vls-core/src/policy/simple_validator.rs` (primary), `onchain_validator.rs` (chain-aware), documented in `docs/policy-controls.md`.

---

## Table of Contents

1. [Channel Opening](#1-channel-opening)
2. [On-chain Transactions](#2-on-chain-transactions)
3. [Commitment Transaction — Format & Structure](#3-commitment-transaction--format--structure)
4. [Commitment Transaction — Value & Fee](#4-commitment-transaction--value--fee)
5. [Commitment Transaction — HTLC Validation](#5-commitment-transaction--htlc-validation)
6. [Commitment Transaction — Key Validation](#6-commitment-transaction--key-validation)
7. [Commitment Transaction — Revocation & State](#7-commitment-transaction--revocation--state)
8. [Commitment Transaction — Anchor Outputs](#8-commitment-transaction--anchor-outputs)
9. [Payment Validation](#9-payment-validation)
10. [Revocation Rules](#10-revocation-rules)
11. [HTLC Transaction Validation](#11-htlc-transaction-validation)
12. [Mutual Close](#12-mutual-close)
13. [Sweep Transactions](#13-sweep-transactions)
14. [Funding Limits](#14-funding-limits)
15. [Velocity Limits](#15-velocity-limits)
16. [Routing Hub Rules](#16-routing-hub-rules)
17. [Merchant Rules](#17-merchant-rules)
18. [Chain Validation](#18-chain-validation)
19. [General](#19-general)

---

## 1. Channel Opening

These rules are enforced when a new channel is being negotiated and set up.

### 1. policy-channel-contest-delay-range-holder

| Field | Value |
|-------|-------|
| **Validates** | The local `to_self_delay` value (contest delay imposed on the holder) is within a valid range |
| **Why it matters** | The contest delay determines how long you must wait to claim your funds after a unilateral close. Too short means the counterparty can steal funds before you detect a breach. Too long means your funds are locked for an unacceptable period. |
| **Attack prevented** | Counterparty negotiating an extremely short delay, then broadcasting a revoked commitment and sweeping funds before the penalty can be applied |
| **Default params** | min=144 blocks (~1 day), max=2016 blocks (~2 weeks) |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 2. policy-channel-contest-delay-range-counterparty

| Field | Value |
|-------|-------|
| **Validates** | The remote/counterparty `to_self_delay` value is within a valid range |
| **Why it matters** | This is the delay imposed on the counterparty. If set too low, it reduces our time to detect and punish cheating. If set too high, it unreasonably locks up counterparty funds (which may indicate a malformed negotiation). |
| **Attack prevented** | Misconfigured delays that reduce the effectiveness of the penalty mechanism |
| **Default params** | min=144 blocks (~1 day), max=2016 blocks (~2 weeks) |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 3. policy-channel-safe-mode

| Field | Value |
|-------|-------|
| **Validates** | The channel commitment type must be a safe, modern type |
| **Why it matters** | Older commitment types (Legacy, plain Anchors) lack security features of modern Lightning. Legacy channels don't support static remote keys. Plain anchor channels have fee-theft vulnerabilities in HTLC transactions. Only `StaticRemoteKey` and `AnchorsZeroFeeHtlc` are considered safe. |
| **Attack prevented** | Downgrade attacks where a malicious peer forces use of an older, less secure channel type |
| **Allowed types** | `StaticRemoteKey`, `AnchorsZeroFeeHtlc` |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 4. policy-channel-original-channel-id-reuse

| Field | Value |
|-------|-------|
| **Validates** | The original channel ID is not reused across different channels |
| **Why it matters** | Channel creation messages may arrive out of order. If a channel ID is reused, the signer could confuse state between two different channels, potentially signing a commitment for the wrong channel and losing funds. |
| **Attack prevented** | Channel confusion attacks where reused IDs cause state mix-ups |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 5. policy-generic-error

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for generic validation errors caused by incorrect user input |
| **Why it matters** | Provides a fallback error category for inputs that don't match any specific rule but are clearly invalid |
| **Type** | Mandatory |
| **Status** | Implemented |

---

## 2. On-chain Transactions

These rules validate funding transactions, wallet transactions, and other on-chain operations.

### 6. policy-onchain-output-scriptpubkey

| Field | Value |
|-------|-------|
| **Validates** | Transaction outputs have correct and standard scriptPubKey format |
| **Why it matters** | A malformed script cannot be spent by anyone — funds sent to it are permanently lost. The signer verifies that the wallet can actually spend from the derived script before signing. |
| **Attack prevented** | Compromised node constructing transactions with unspendable outputs, burning funds |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 7. policy-onchain-output-match-commitment

| Field | Value |
|-------|-------|
| **Validates** | The funding output value matches what the initial commitment transaction expects |
| **Why it matters** | A mismatch between the on-chain funding amount and the commitment transaction's expected input value would make the commitment transaction invalid and unspendable. |
| **Attack prevented** | Node constructing a funding transaction with wrong value, making the channel unusable and funds stuck |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 8. policy-onchain-initial-commitment-countersigned

| Field | Value |
|-------|-------|
| **Validates** | Before signing a funding transaction, the initial holder commitment must already be countersigned |
| **Why it matters** | If you fund a channel without having a valid commitment transaction ready, the counterparty could refuse to provide one, and your funds would be locked in a 2-of-2 multisig with no way to recover them. |
| **Attack prevented** | Counterparty baiting you into funding a channel, then holding funds hostage by never providing a signed commitment |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 9. policy-onchain-no-unknown-outputs

| Field | Value |
|-------|-------|
| **Validates** | Every output in an on-chain transaction must be accounted for: either going to our wallet, an allowlisted destination, or funding a known channel |
| **Why it matters** | Unknown outputs mean funds are being sent somewhere uncontrolled. A compromised node could add extra outputs to steal funds during what appears to be a normal wallet transaction. |
| **Attack prevented** | Compromised node adding theft outputs to funding or sweep transactions |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 10. policy-onchain-no-channel-push

| Field | Value |
|-------|-------|
| **Validates** | Funding transactions cannot include a `push_msat` value (giving initial balance to counterparty) |
| **Why it matters** | Push value means you fund the channel but immediately give some balance to the counterparty. A compromised node could push the entire funding amount to the peer, effectively stealing all funds. |
| **Attack prevented** | Compromised node gifting channel funds to a colluding counterparty |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 11. policy-onchain-no-fund-inbound

| Field | Value |
|-------|-------|
| **Validates** | Cannot sign funding transactions for inbound channels (channels where we don't initiate funding) |
| **Why it matters** | In the current protocol (pre-dual-funding), only the channel opener funds the channel. Signing a funding tx for an inbound channel would be sending funds into a channel we didn't open, which makes no sense and could lose funds. |
| **Attack prevented** | Tricking the signer into funding a channel opened by someone else |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 12. policy-onchain-wallet-path-predictable

| Field | Value |
|-------|-------|
| **Validates** | Change output derivation paths must be within a predictable range |
| **Why it matters** | If change goes to an unexpected derivation path, wallet software may not find it during scanning, effectively losing the funds. A compromised node could derive to a path only the attacker knows. |
| **Attack prevented** | Directing change to attacker-controlled derivation paths |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 13. policy-onchain-fee-range

| Field | Value |
|-------|-------|
| **Validates** | Transaction fees must be within acceptable bounds (min and max sat/kw) |
| **Why it matters** | Excessively high fees are economic loss (possibly to a colluding miner). Insufficient fees may prevent transaction confirmation, keeping funds stuck. |
| **Attack prevented** | Fee-siphoning attack where a compromised node sets enormous fees, effectively burning funds or directing them to a miner co-conspirator |
| **Default params** | min=253 sat/kw, max=25,000 sat/kw |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 14. policy-onchain-format-standard

| Field | Value |
|-------|-------|
| **Validates** | Transaction fields must be standard (version=2, etc.) |
| **Why it matters** | Non-standard transactions may not be relayed by the Bitcoin network or included in blocks, leaving funds permanently stuck. |
| **Attack prevented** | Constructing non-relayable transactions that trap funds |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 15. policy-onchain-funding-non-malleable

| Field | Value |
|-------|-------|
| **Validates** | Funding transactions must use only SegWit inputs (no legacy P2PKH or P2SH inputs) |
| **Why it matters** | Non-SegWit inputs can have their witness data malleated by third parties, changing the transaction ID. Since the commitment transaction references the funding txid, malleation would invalidate all commitment transactions, permanently locking channel funds. |
| **Attack prevented** | Transaction malleation attack that breaks the funding-to-commitment chain |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 16. policy-onchain-max-size

| Field | Value |
|-------|-------|
| **Validates** | Transaction serialized size must not exceed 32 KB |
| **Why it matters** | Bitcoin has practical limits on transaction size for relay and block inclusion. Oversized transactions won't propagate through the network. |
| **Attack prevented** | Constructing unreasonably large transactions that cannot confirm |
| **Default params** | max=32,768 bytes |
| **Type** | Mandatory |
| **Status** | Fully implemented |

---

## 3. Commitment Transaction — Format & Structure

These rules ensure commitment transactions conform to BOLT 3 specification.

### 17. policy-commitment-version

| Field | Value |
|-------|-------|
| **Validates** | Commitment transaction version must be 2 |
| **Why it matters** | Version 2 is required for relative timelocks (BIP 68/112) which are essential for Lightning's revocation mechanism. Version 1 transactions cannot use CSV (CheckSequenceVerify). |
| **Attack prevented** | Downgrade to version 1 that would disable timelock-based penalty enforcement |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 18. policy-commitment-locktime

| Field | Value |
|-------|-------|
| **Validates** | Locktime must be encoded per BOLT 3 (encodes obscured commitment number) |
| **Why it matters** | BOLT 3 encodes the lower 24 bits of the obscured commitment number in the locktime field. Incorrect locktime means the commitment number tracking is broken, which could allow replaying old states. |
| **Attack prevented** | State number manipulation |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 19. policy-commitment-sequence

| Field | Value |
|-------|-------|
| **Validates** | Input sequence numbers must be correct per BOLT 3 (encodes upper bits of obscured commitment number) |
| **Why it matters** | The upper 24 bits of the obscured commitment number are encoded in the sequence field. This is part of the mechanism that allows breach detection and penalty transaction construction. |
| **Attack prevented** | Tampering with commitment number encoding |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 20. policy-commitment-scripts

| Field | Value |
|-------|-------|
| **Validates** | All output scripts must match expected BOLT 3 templates (to-local, to-remote, HTLC-offered, HTLC-received) |
| **Why it matters** | Scripts define the spending conditions. A modified script could allow the counterparty to spend our funds without proper authorization, or could make outputs unspendable entirely. |
| **Attack prevented** | Script substitution that redirects funds or locks them permanently |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 21. policy-commitment-other

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for other commitment transaction format requirements |
| **Why it matters** | Covers miscellaneous format violations not captured by specific rules |
| **Type** | Mandatory |
| **Status** | Implemented |

### 22. policy-commitment-input-single

| Field | Value |
|-------|-------|
| **Validates** | Commitment transaction must have exactly one input |
| **Why it matters** | Per BOLT 3, a commitment transaction always spends exactly the single funding output. Multiple inputs would indicate a malformed or fraudulent transaction that doesn't correspond to any valid channel state. |
| **Attack prevented** | Injecting extra inputs to manipulate transaction structure |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 23. policy-commitment-input-match-funding

| Field | Value |
|-------|-------|
| **Validates** | The single input must reference the correct funding outpoint (txid:vout) |
| **Why it matters** | A commitment that spends the wrong UTXO is either invalid (won't confirm) or is spending someone else's funds. The signer must verify the input matches the channel's known funding output. |
| **Attack prevented** | Tricking the signer into signing a transaction that spends the wrong UTXO |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 24. policy-commitment-singular-to-holder

| Field | Value |
|-------|-------|
| **Validates** | At most one to-local (holder) output in the commitment transaction |
| **Why it matters** | Per BOLT 3, there is exactly zero or one to-local output. Multiple would indicate a malformed transaction and could confuse state tracking. |
| **Attack prevented** | Malformed commitment structure that breaks state assumptions |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 25. policy-commitment-singular-to-counterparty

| Field | Value |
|-------|-------|
| **Validates** | At most one to-remote (counterparty) output in the commitment transaction |
| **Why it matters** | Same as above — per BOLT 3, at most one to-remote output exists. Multiple would indicate structural corruption. |
| **Attack prevented** | Malformed commitment structure |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 26. policy-commitment-no-unrecognized-outputs

| Field | Value |
|-------|-------|
| **Validates** | All outputs must be identifiable as: to-local, to-remote, offered-HTLC, received-HTLC, or anchor |
| **Why it matters** | Any unrecognized output means funds are going somewhere unknown. This could be theft, or it could mean the signer cannot properly account for all channel funds. |
| **Attack prevented** | Injecting extra outputs to siphon channel funds |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 27. policy-commitment-outputs-trimmed

| Field | Value |
|-------|-------|
| **Validates** | Outputs below the dust limit must be properly trimmed (omitted from the transaction) |
| **Why it matters** | Dust outputs (below ~546 sat for P2WSH) cannot be economically spent due to fees exceeding their value. Per BOLT 3, such outputs must be omitted and their value added to fees. Including them wastes chain space and creates unspendable UTXOs. |
| **Attack prevented** | Creating unspendable dust outputs that effectively burn funds |
| **Type** | Mandatory |
| **Status** | Fully implemented |

---

## 4. Commitment Transaction — Value & Fee

### 28. policy-commitment-initial-funding-value

| Field | Value |
|-------|-------|
| **Validates** | In the initial commitment (commit_num=0), if we are the funder, our balance must equal the funded amount (minus any push_value) |
| **Why it matters** | The first commitment must accurately reflect the funded amount. If the initial state gives us less than we funded, we've been cheated from the very start. |
| **Attack prevented** | Counterparty constructing initial commitment that steals part of our funding |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 29. policy-commitment-fee-range

| Field | Value |
|-------|-------|
| **Validates** | Commitment transaction fee must be within acceptable feerate bounds |
| **Why it matters** | The fee is implicitly paid by the funder (difference between input value and total output value). Excessive fees drain the funder's balance. Insufficient fees risk the commitment never confirming if broadcast. |
| **Attack prevented** | Fee manipulation to drain channel balance or create unconfirmable commitments |
| **Default params** | min=253 sat/kw, max=25,000 sat/kw |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 30. policy-commitment-spends-active-utxo

| Field | Value |
|-------|-------|
| **Validates** | The funding UTXO must be confirmed on-chain with sufficient depth |
| **Why it matters** | If the funding output was double-spent or never confirmed, the entire channel is invalid. Signing commitments for a non-existent channel is pointless and could be used to manipulate state tracking. |
| **Attack prevented** | Operating on phantom channels backed by non-existent funding |
| **Type** | Mandatory |
| **Status** | Not yet implemented (requires UTXO oracle integration) |

---

## 5. Commitment Transaction — HTLC Validation

### 31. policy-commitment-first-no-htlcs

| Field | Value |
|-------|-------|
| **Validates** | The initial commitment transaction (commit_num=0) must contain no HTLC outputs |
| **Why it matters** | Before the channel is operational, there should be no in-flight payments. HTLCs in the first commitment indicate a protocol violation or attack attempt. |
| **Attack prevented** | Injecting fake HTLCs into the initial state to steal funds |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 32. policy-commitment-htlc-inflight-limit

| Field | Value |
|-------|-------|
| **Validates** | The total value of all in-flight HTLCs must not exceed the configured maximum |
| **Why it matters** | In-flight HTLCs represent funds at risk — if the channel force-closes with pending HTLCs, the resolution is complex and potentially lossy. Capping the total limits maximum exposure. |
| **Attack prevented** | Flooding the channel with maximum-value HTLCs to maximize potential loss in a force-close |
| **Default params** | max=16,777,216 sat |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 33. policy-commitment-htlc-count-limit

| Field | Value |
|-------|-------|
| **Validates** | The number of concurrent HTLC outputs must not exceed the maximum |
| **Why it matters** | Each HTLC adds an output to the commitment transaction, increasing its size and the complexity of force-close resolution. Too many HTLCs also increase on-chain fees required to resolve them all. |
| **Attack prevented** | HTLC spam attack that makes force-close resolution prohibitively expensive |
| **Default params** | max=1,000 HTLCs |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 34. policy-commitment-htlc-routing-balance

| Field | Value |
|-------|-------|
| **Validates** | Each offered HTLC (outgoing payment) must be balanced by a corresponding received HTLC (incoming payment) of equal or greater value |
| **Why it matters** | For routing nodes, you should never offer more value than you receive — that's paying out of pocket. This ensures the node cannot be tricked into forwarding payments it hasn't received funding for. |
| **Attack prevented** | Compromised node routing payments without corresponding incoming funds, draining the channel |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 35. policy-commitment-htlc-cltv-range

| Field | Value |
|-------|-------|
| **Validates** | The CLTV expiry on received HTLC outputs must be reasonable (not expired, not too far in future) |
| **Why it matters** | An already-expired HTLC means the counterparty can immediately claim it back. An expiry too far in the future locks funds for an excessive period. Both represent economic harm. |
| **Attack prevented** | Receiving HTLCs with manipulated timelocks that guarantee loss |
| **Default params** | Disabled by default; enabled with `use_chain_state=true`; max absolute=500,000,000 |
| **Type** | Mandatory |
| **Status** | Partially implemented (absolute max only; relative check requires chain state) |

### 36. policy-commitment-htlc-received-spends-active-utxo

| Field | Value |
|-------|-------|
| **Validates** | For received HTLCs, the funding UTXO of the receiving channel must be active on-chain |
| **Why it matters** | If the incoming channel's funding has been spent (channel closed), the received HTLC can never be claimed on-chain. Forwarding based on a dead channel loses funds. |
| **Attack prevented** | Routing through channels with already-spent funding outputs |
| **Type** | Mandatory |
| **Status** | Not yet implemented (requires UTXO oracle) |

### 37. policy-commitment-htlc-offered-hash-matches

| Field | Value |
|-------|-------|
| **Validates** | The payment hash of an offered HTLC must match a corresponding received HTLC |
| **Why it matters** | In routing, the offered and received HTLCs must share the same payment hash so that revealing the preimage settles both atomically. Mismatched hashes break this atomicity. |
| **Attack prevented** | Hash mismatch that prevents atomic settlement, potentially losing routed funds |
| **Type** | Mandatory |
| **Status** | Not yet implemented (may be redundant with routing-balance check) |

---

## 6. Commitment Transaction — Key Validation

These rules verify that all cryptographic keys in commitment outputs are correctly derived.

### 38. policy-commitment-revocation-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The revocation public key in commitment outputs is correctly derived from both parties' basepoints |
| **Why it matters** | The revocation key is what allows punishment of cheaters. If it's wrong, broadcasting a revoked commitment cannot be penalized — the entire security model breaks. |
| **Attack prevented** | Substituting a wrong revocation key that prevents penalty enforcement |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 39. policy-commitment-to-self-delay-range

| Field | Value |
|-------|-------|
| **Validates** | The `to_self_delay` encoded in the to-local output script matches the negotiated value |
| **Why it matters** | The delay gives the other party time to detect and punish a breach. If the actual script uses a different (shorter) delay than negotiated, the cheater could sweep funds before detection. |
| **Attack prevented** | Reducing the effective contest delay below what was agreed |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 40. policy-commitment-broadcaster-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The delayed payment pubkey in the to-local output is correctly derived |
| **Why it matters** | This is the key that allows us to spend our own funds after the CSV delay. A wrong key means we can never claim our channel balance after a unilateral close. |
| **Attack prevented** | Key substitution that makes our funds permanently unspendable |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 41. policy-commitment-countersignatory-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The remote pubkey in the to-remote output is correctly derived |
| **Why it matters** | This key controls the counterparty's immediate-spend output. While less critical for us, an incorrect key could indicate broader commitment corruption. |
| **Attack prevented** | Commitment structure corruption |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 42. policy-commitment-htlc-revocation-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The revocation pubkey embedded in each HTLC output script is correct |
| **Why it matters** | If the counterparty broadcasts a revoked commitment, we need to sweep not just the to-local output but also all HTLC outputs using the revocation key. Wrong HTLC revocation keys mean those HTLCs cannot be penalized. |
| **Attack prevented** | Partial penalty evasion by using wrong keys only in HTLC outputs |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 43. policy-commitment-htlc-counterparty-htlc-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The counterparty's HTLC pubkeys (derived from per-commitment point) are correct |
| **Why it matters** | HTLC scripts require signatures from both parties' HTLC keys. Wrong counterparty keys would make HTLCs unresolvable. |
| **Attack prevented** | HTLC key substitution that locks HTLC funds permanently |
| **Type** | Mandatory |
| **Status** | Implemented (tests not yet complete) |

### 44. policy-commitment-htlc-holder-htlc-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The holder's HTLC pubkeys are correctly derived |
| **Why it matters** | Same as above — both parties' keys must be correct for HTLC scripts to function |
| **Attack prevented** | HTLC key corruption |
| **Type** | Mandatory |
| **Status** | Implemented (tests not yet complete) |

---

## 7. Commitment Transaction — Revocation & State

### 45. policy-commitment-previous-revoked

| Field | Value |
|-------|-------|
| **Validates** | Before accepting a new commitment, the previous one must have been properly revoked by the counterparty disclosing their revocation secret |
| **Why it matters** | Revocation secrets enable penalty transactions. Without receiving the revocation for state N, moving to state N+1 means state N could be broadcast without penalty. The signer validates secret consistency using BOLT 3's compact storage scheme. |
| **Attack prevented** | Advancing state without proper revocation, leaving old states unpunishable |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 46. policy-commitment-holder-not-revoked

| Field | Value |
|-------|-------|
| **Validates** | We must not sign a local commitment that we have already revoked |
| **Why it matters** | If we broadcast a revoked commitment, the counterparty can claim ALL channel funds via the penalty mechanism. The signer must refuse to sign any commitment we've disclosed the revocation secret for. |
| **Attack prevented** | Compromised node tricking us into signing (and broadcasting) a revoked state, allowing counterparty to take everything |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 47. policy-commitment-retry-same

| Field | Value |
|-------|-------|
| **Validates** | If the same commitment number is presented again (retry), the data must be identical to the first attempt |
| **Why it matters** | A retry with different data could indicate an attempt to get signatures for two different transactions at the same commitment number — effectively creating two valid states, breaking the revocation model. |
| **Attack prevented** | Double-signing different transactions at the same state number |
| **Type** | Mandatory |
| **Status** | Fully implemented |

---

## 8. Commitment Transaction — Anchor Outputs

These rules validate anchor outputs used for CPFP (Child Pays For Parent) fee bumping.

### 48. policy-commitment-anchors-not-when-off

| Field | Value |
|-------|-------|
| **Validates** | If neither anchor option is negotiated, no anchor outputs should exist |
| **Why it matters** | Unexpected anchor outputs in a non-anchor channel waste funds (330 sat per anchor) and violate the protocol |
| **Attack prevented** | Injecting unexpected outputs disguised as anchors |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 49. policy-commitment-anchor-to-holder

| Field | Value |
|-------|-------|
| **Validates** | When anchors are enabled and a to-local output exists, the holder's anchor output must be present |
| **Why it matters** | The anchor allows the holder to fee-bump the commitment transaction after broadcast. Without it, a low-fee commitment might never confirm, trapping funds. |
| **Attack prevented** | Omitting the anchor output, preventing fee-bumping capability |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 50. policy-commitment-anchor-to-counterparty

| Field | Value |
|-------|-------|
| **Validates** | When anchors are enabled and a to-remote output exists, the counterparty's anchor output must be present |
| **Why it matters** | Both parties need fee-bumping capability for the commitment to be useful in all fee environments |
| **Attack prevented** | Selectively omitting counterparty's anchor |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 51. policy-commitment-anchor-amount

| Field | Value |
|-------|-------|
| **Validates** | Anchor outputs must be exactly the correct amount (330 sat) |
| **Why it matters** | Anchors at incorrect amounts either waste funds (too high) or are unspendable as dust (too low) |
| **Attack prevented** | Manipulating anchor values |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 52. policy-commitment-anchor-static-remotekey

| Field | Value |
|-------|-------|
| **Validates** | When using anchors, `option_static_remotekey` must be enabled |
| **Why it matters** | The anchor specification requires static remote keys. Without this, key derivation is incompatible with the anchor spending mechanism. |
| **Attack prevented** | Feature mismatch that breaks anchor functionality |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 53. policy-commitment-anchor-match-fundingkey

| Field | Value |
|-------|-------|
| **Validates** | Each anchor output must be locked by the correct party's funding key |
| **Why it matters** | If an anchor is locked by the wrong key, the intended party cannot spend it for fee-bumping |
| **Attack prevented** | Anchor key substitution |
| **Type** | Mandatory |
| **Status** | Not yet implemented (tests not complete) |

---

## 9. Payment Validation

These rules validate outgoing payments and invoices.

### 54. policy-commitment-payment-settled-preimage

| Field | Value |
|-------|-------|
| **Validates** | When an offered HTLC is being settled (removed with preimage), the preimage must be known to us |
| **Why it matters** | If we don't know the preimage for a payment we're routing, we cannot claim the incoming HTLC. This would mean we paid out without being able to collect — a net loss. |
| **Attack prevented** | Settling payments with unknown preimages, causing routing loss |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 55. policy-commitment-payment-allowlisted

| Field | Value |
|-------|-------|
| **Validates** | Outgoing payments must be directed to allowlisted destinations |
| **Why it matters** | Restricts which nodes/destinations can receive payments, providing defense-in-depth against compromised nodes making unauthorized payments |
| **Attack prevented** | Compromised node sending payments to attacker-controlled nodes |
| **Type** | Optional |
| **Status** | Not yet implemented |

### 56. policy-commitment-payment-velocity

| Field | Value |
|-------|-------|
| **Validates** | Total outgoing payment amount must not exceed a velocity limit per time period |
| **Why it matters** | Even if a node is compromised, limiting the rate of outgoing payments caps the total damage. The attacker can only steal up to the velocity limit before the compromise is detected. |
| **Attack prevented** | Rapid fund drainage from a compromised node |
| **Type** | Optional |
| **Status** | Implemented (as warning in `add_invoice` phase) |

### 57. policy-commitment-payment-approved

| Field | Value |
|-------|-------|
| **Validates** | Outgoing payment must be explicitly approved via an out-of-band mechanism |
| **Why it matters** | Provides a human-in-the-loop or external system approval gate for payments, adding a second factor beyond just the node's request |
| **Attack prevented** | Unauthorized payments from a compromised node without external verification |
| **Type** | Optional |
| **Status** | Not yet implemented |

### 58. policy-commitment-payment-invoiced

| Field | Value |
|-------|-------|
| **Validates** | The amount sent must not exceed the invoice amount |
| **Why it matters** | Overpaying an invoice is a direct economic loss. A compromised node could create invoices for large amounts and route payments to itself. |
| **Attack prevented** | Overpayment attacks and invoice inflation |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 59. policy-invoice-not-expired

| Field | Value |
|-------|-------|
| **Validates** | Invoice must not have passed its expiry time |
| **Why it matters** | Expired invoices may have been cancelled, replaced, or issued in error. Paying an expired invoice risks funds being unclaimable by the recipient (if they've deleted the preimage). |
| **Attack prevented** | Paying stale invoices that may no longer be valid |
| **Type** | Mandatory |
| **Status** | Fully implemented |

---

## 10. Revocation Rules

### 60. policy-revoke-new-commitment-signed

| Field | Value |
|-------|-------|
| **Validates** | Before revoking the old commitment, the counterparty must have signed the new one |
| **Why it matters** | If you revoke state N without having a valid state N+1, you have no usable commitment transaction. You'd be unable to force-close, leaving you entirely dependent on counterparty cooperation. |
| **Attack prevented** | Tricking the signer into revoking without having a replacement state, making force-close impossible |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 61. policy-revoke-new-commitment-valid

| Field | Value |
|-------|-------|
| **Validates** | The new commitment transaction (that will become current after revocation) must pass all policy checks |
| **Why it matters** | Revoking old state and moving to an invalid new state leaves you with no valid commitment. The new state must be fully validated before the old one is discarded. |
| **Attack prevented** | Advancing to an invalid state that cannot be broadcast |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 62. policy-revoke-not-closed

| Field | Value |
|-------|-------|
| **Validates** | Cannot revoke a commitment if a closing transaction has already been signed |
| **Why it matters** | Once a mutual close is signed, the channel should be in a terminal state. Revoking commitments after close introduces inconsistency. |
| **Attack prevented** | State manipulation after channel close |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

---

## 11. HTLC Transaction Validation

These rules validate second-level HTLC transactions (HTLC-success and HTLC-timeout transactions that spend HTLC outputs from commitment transactions).

### 63. policy-htlc-version

| Field | Value |
|-------|-------|
| **Validates** | HTLC transaction version must be 2 |
| **Why it matters** | Version 2 is required for relative timelocks (CSV) used in HTLC output spending conditions |
| **Attack prevented** | Version downgrade disabling CSV enforcement |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 64. policy-htlc-locktime

| Field | Value |
|-------|-------|
| **Validates** | For HTLC-timeout transactions, locktime must equal the HTLC's CLTV expiry. For HTLC-success, locktime must be 0. |
| **Why it matters** | The locktime determines when the HTLC can be timed out. A wrong locktime could allow premature timeout (stealing from receiver) or prevent timeout entirely (locking funds). |
| **Attack prevented** | Locktime manipulation to steal HTLC funds or lock them permanently |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 65. policy-htlc-sequence

| Field | Value |
|-------|-------|
| **Validates** | Input sequence must be correct per BOLT 3 (enables relative timelock) |
| **Why it matters** | Sequence enables the CSV delay on HTLC transaction outputs, giving the revocation mechanism time to work |
| **Attack prevented** | Disabling the CSV delay on HTLC outputs |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 66. policy-htlc-other

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for other HTLC transaction format requirements |
| **Why it matters** | Covers miscellaneous HTLC format violations |
| **Type** | Mandatory |
| **Status** | Implemented |

### 67. policy-htlc-cltv-range

| Field | Value |
|-------|-------|
| **Validates** | HTLC timeout locktime (cltv_expiry) must be reasonable relative to current chain height |
| **Why it matters** | An already-passed CLTV means the timeout path is immediately spendable, potentially allowing counterparty to take funds. A too-far-future CLTV locks funds excessively. |
| **Attack prevented** | Manipulated CLTV values that enable immediate theft or indefinite locking |
| **Type** | Mandatory |
| **Status** | Not yet implemented |

### 68. policy-htlc-revocation-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The revocation pubkey in the HTLC transaction output must be correct |
| **Why it matters** | The HTLC transaction output is spendable via revocation if a breach occurs. Wrong revocation key means this penalty path is broken for HTLC-level transactions. |
| **Attack prevented** | Disabling penalty enforcement at the HTLC transaction level |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 69. policy-htlc-to-self-delay

| Field | Value |
|-------|-------|
| **Validates** | The CSV delay in the HTLC transaction output must match the negotiated `to_self_delay` |
| **Why it matters** | This delay gives the counterparty time to detect and penalize a breach at the HTLC level. A shorter delay reduces the detection window. |
| **Attack prevented** | Reducing the HTLC-level penalty detection window |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 70. policy-htlc-delayed-pubkey

| Field | Value |
|-------|-------|
| **Validates** | The delayed payment pubkey in the HTLC transaction output must be correctly derived |
| **Why it matters** | This key controls spending after the CSV delay. Wrong key means we cannot claim our HTLC transaction output. |
| **Attack prevented** | Key substitution that makes HTLC proceeds unspendable by us |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 71. policy-htlc-fee-range

| Field | Value |
|-------|-------|
| **Validates** | HTLC transaction fee must be within acceptable bounds |
| **Why it matters** | Excessive fees on HTLC transactions drain the HTLC value. For pre-anchor channels, HTLC fees are fixed at commitment time — too-high fees reduce the claimable HTLC amount. |
| **Attack prevented** | Fee manipulation to reduce HTLC value |
| **Default params** | min=253 sat/kw, max=25,000 sat/kw |
| **Type** | Mandatory |
| **Status** | Fully implemented |

---

## 12. Mutual Close

These rules validate cooperative (mutual) close transactions.

### 72. policy-mutual-destination-allowlisted

| Field | Value |
|-------|-------|
| **Validates** | The close destination must be in our wallet or on the allowlist. If an `upfront_shutdown_script` was specified at channel open, the close must go there. |
| **Why it matters** | A compromised node could direct close proceeds to an attacker-controlled address. The allowlist ensures funds only go to pre-approved destinations. |
| **Attack prevented** | Redirecting close proceeds to attacker's address |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 73. policy-mutual-value-matches-commitment

| Field | Value |
|-------|-------|
| **Validates** | Our output value in the closing transaction must match our balance in the latest commitment |
| **Why it matters** | The counterparty could propose a close transaction that gives us less than our actual balance, stealing the difference. The signer verifies the amounts match the known state. |
| **Attack prevented** | Counterparty cheating on the mutual close amount |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 74. policy-mutual-fee-range

| Field | Value |
|-------|-------|
| **Validates** | The mutual close transaction fee must be within acceptable bounds |
| **Why it matters** | Excessive close fees drain our balance. Since both parties must agree on the fee, a compromised node might agree to an unreasonable fee proposed by the counterparty. |
| **Attack prevented** | Fee manipulation during cooperative close |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 75. policy-mutual-no-pending-htlcs

| Field | Value |
|-------|-------|
| **Validates** | Cannot perform a mutual close while there are unresolved HTLCs |
| **Why it matters** | Pending HTLCs represent funds in transit. A mutual close with pending HTLCs would lose those funds — they can't be resolved once the channel is closed cooperatively. |
| **Attack prevented** | Closing a channel while HTLCs are pending, causing those funds to be lost |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 76. policy-mutual-other

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for other mutual close format requirements |
| **Why it matters** | Covers miscellaneous format violations |
| **Type** | Mandatory |
| **Status** | Implemented |

### 77. policy-mutual-scripts

| Field | Value |
|-------|-------|
| **Validates** | Close output scripts must be valid and standard |
| **Why it matters** | Non-standard scripts may be unspendable or not accepted by the network |
| **Attack prevented** | Malformed scripts that trap close proceeds |
| **Type** | Mandatory |
| **Status** | Implemented |

---

## 13. Sweep Transactions

These rules validate transactions that sweep funds back to the wallet after a channel close (e.g., spending the to-local output after CSV delay, or spending HTLC outputs).

### 78. policy-sweep-version

| Field | Value |
|-------|-------|
| **Validates** | Sweep transaction version must be 2 |
| **Why it matters** | Required for relative timelocks used in spending delayed outputs |
| **Attack prevented** | Version downgrade |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 79. policy-sweep-sequence

| Field | Value |
|-------|-------|
| **Validates** | Input sequence must be valid for the type of sweep |
| **Why it matters** | Different sweep types require different sequence values. For anchor channels: `0x00000001` (1 block CSV). For non-anchor: `0x00000000`, `0xfffffffd`, or `0xffffffff` depending on context. Wrong sequence could make the sweep invalid. |
| **Attack prevented** | Invalid sequence that either fails validation or bypasses intended timelocks |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 80. policy-sweep-locktime

| Field | Value |
|-------|-------|
| **Validates** | Locktime must not be too far in the future (current height + MAX_CHAIN_LAG of 2 blocks) |
| **Why it matters** | A too-far-future locktime delays fund recovery unnecessarily. It should be at or near the current block height. |
| **Attack prevented** | Artificially delaying fund recovery |
| **Default params** | max = current_height + 2 |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 81. policy-sweep-destination-allowlisted

| Field | Value |
|-------|-------|
| **Validates** | Sweep outputs must go to our wallet or an allowlisted destination |
| **Why it matters** | Swept funds should return to our control. A compromised node could direct sweeps to attacker addresses. |
| **Attack prevented** | Redirecting swept channel funds to attacker |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 82. policy-sweep-fee-range

| Field | Value |
|-------|-------|
| **Validates** | Sweep transaction fee must be reasonable |
| **Why it matters** | Excessive sweep fees drain recovered funds. Since sweeps happen after force-close (often urgent), there's temptation to overpay. |
| **Attack prevented** | Fee inflation on sweep transactions |
| **Type** | Mandatory |
| **Status** | Not yet implemented (blocked by multi-input sweep complexity in LDK) |

### 83. policy-sweep-other

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for other sweep format requirements |
| **Why it matters** | Covers miscellaneous sweep format violations |
| **Type** | Mandatory |
| **Status** | Implemented |

---

## 14. Funding Limits

### 84. policy-funding-max

| Field | Value |
|-------|-------|
| **Validates** | A single channel's funded amount must not exceed the configured maximum |
| **Why it matters** | Caps maximum loss per channel. Even with all other protections, limiting channel size bounds the worst case. |
| **Attack prevented** | Concentrating excessive funds in a single channel |
| **Default params** | max=1,000,000,001 sat (~10 BTC) |
| **Type** | Optional |
| **Status** | Fully implemented |

### 85. policy-velocity-funding

| Field | Value |
|-------|-------|
| **Validates** | Total amount funded across all channels per time period must not exceed a limit |
| **Why it matters** | Even if individual channels are capped, opening many channels rapidly could drain the wallet. A velocity limit on funding operations prevents rapid depletion. |
| **Attack prevented** | Rapidly opening many max-size channels to drain wallet |
| **Type** | Optional |
| **Status** | Not yet implemented |

---

## 15. Velocity Limits

### 86. policy-velocity-transferred

| Field | Value |
|-------|-------|
| **Validates** | Total amount transferred to peers per time period must not exceed a configured limit |
| **Why it matters** | The ultimate rate-limit on fund loss. Even with a fully compromised node, the attacker can only extract funds at the velocity rate. This gives operators time to detect and respond. |
| **Attack prevented** | Sustained fund drainage over time from a compromised node |
| **Type** | Optional |
| **Status** | Not yet implemented |

---

## 16. Routing Hub Rules

These rules are specific to nodes operating as routing hubs.

### 87. policy-routing-balanced

| Field | Value |
|-------|-------|
| **Validates** | Total amount claimable on output channels must not exceed amount claimable on input channels |
| **Why it matters** | A routing node should never pay out more than it receives. This is the fundamental invariant of routing — you forward payments, not subsidize them. |
| **Attack prevented** | Routing at a loss, draining channel balances |
| **Type** | Mandatory (for routing hubs) |
| **Status** | Fully implemented |

### 88. policy-routing-cltv-delta

| Field | Value |
|-------|-------|
| **Validates** | The CLTV delta between incoming and outgoing HTLCs must be sufficient |
| **Why it matters** | The CLTV delta is the time window for claiming the incoming HTLC after the outgoing one settles. Too small a delta means the incoming HTLC could expire before we claim it, losing the routed amount. |
| **Attack prevented** | Timing attack where outgoing HTLC succeeds but incoming expires before we can claim |
| **Default params** | min=34 blocks (accounts for 3 retries + grace period) |
| **Type** | Mandatory |
| **Status** | Fully implemented |

### 89. policy-routing-deltas-only-htlc

| Field | Value |
|-------|-------|
| **Validates** | Channel balance changes must only occur through HTLC settlement (not direct balance transfers) |
| **Why it matters** | For a routing hub, the only legitimate way balances change is through HTLC forwarding. Any other balance change indicates something abnormal — either a bug or an attack. |
| **Attack prevented** | Non-HTLC balance manipulation in routing channels |
| **Type** | Optional (routing hub specific) |
| **Status** | Implemented |

---

## 17. Merchant Rules

### 90. policy-merchant-no-sends

| Field | Value |
|-------|-------|
| **Validates** | Channel balances must only increase (no outbound payments) until channel close or loop-out |
| **Why it matters** | A merchant should only receive payments. If the node is compromised, it shouldn't be able to send funds out — only receive them. This dramatically reduces the attack surface for merchant deployments. |
| **Attack prevented** | Compromised merchant node sending received funds to attacker |
| **Type** | Optional (merchant use-case) |
| **Status** | Not yet implemented |

---

## 18. Chain Validation

### 91. policy-chain-validated

| Field | Value |
|-------|-------|
| **Validates** | Block header attestation provided to the signer must be valid |
| **Why it matters** | The signer uses chain state (block height, UTXO status) for some validations. If an attacker can feed it fake chain data, they could bypass time-based rules. This ensures the chain data is properly attested. |
| **Attack prevented** | Feeding fake blockchain data to the signer to bypass time-based checks |
| **Type** | Mandatory |
| **Status** | Implemented |

---

## 19. General

### 92. policy-other

| Field | Value |
|-------|-------|
| **Validates** | Catch-all for miscellaneous policy requirements not covered by specific rules |
| **Why it matters** | Provides a category for edge cases and new validations being developed |
| **Type** | Mandatory |
| **Status** | Implemented |

---

## Implementation Status Summary

| Status | Count | Percentage |
|--------|-------|------------|
| Fully implemented | ~62 | 67% |
| Partially implemented | ~8 | 9% |
| Not yet implemented | ~17 | 18% |
| Implemented (tests incomplete) | ~5 | 6% |

### Key blockers for unimplemented rules:

- **UTXO oracle** — Rules 30, 36 require on-chain UTXO verification
- **Anchor feature support** — Rules 48-53 need anchor output tree handling
- **Multi-input sweep complexity** — Rule 82 blocked by LDK sweep architecture
- **External approval mechanism** — Rule 57 needs out-of-band approval protocol

---

## Security Model Summary

The policy rules protect against these threat categories:

| Threat | Rules that defend |
|--------|-------------------|
| **Compromised node sending funds to attacker** | 9, 10, 34, 55, 56, 57, 58, 72, 81, 86, 90 |
| **Fee manipulation (draining via fees)** | 13, 29, 71, 74, 82 |
| **Key substitution (making funds unspendable)** | 38-44, 68, 70 |
| **State manipulation (replaying old states)** | 45, 46, 47, 60, 61 |
| **Transaction malleation** | 15, 17-19 |
| **Script corruption** | 6, 20, 27, 77 |
| **HTLC attacks (timing, spam, value)** | 31-37, 63-67, 75, 88 |
| **Channel setup attacks** | 1-4, 8, 10, 11, 28 |
| **Dust/size attacks** | 16, 27, 33 |
| **Fake chain data** | 30, 80, 91 |

---

*Source: [VLS Repository](https://gitlab.com/lightning-signer/validating-lightning-signer), `vls-core/src/policy/` and `docs/policy-controls.md`*
