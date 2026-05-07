# Scenario Roadmap

All scenarios that need to be documented across the epics. Each scenario gets the same numbered filename in reference/, current/, and future/.

---

## Happy Path Scenarios (01–999)

| # | Name | Description | VLS relevance |
|---|------|-------------|---------------|
| 01 | Alice pays Bob | Direct channel: open → HTLC add → HTLC settle → cooperative close | Baseline: all core signer calls in simplest form |
| 02 | Bob routes Alice→Carol | Bob is routing node with two channels, forwards HTLC | Core VLS use case — signer coordinates across two channels |
| 03 | Concurrent HTLCs | Multiple in-flight HTLCs in both directions | Signer state tracking with overlapping commitments |
| 04 | Fee update | Funder sends `update_fee`, new commitment reflects new feerate | Signer must validate fee is reasonable (policy) |
| 05 | Dual-funded open | Both sides contribute to funding tx | Different setup flow — both sign funding, changes channel open calls |
| 06 | Splice in/out | Add or remove funds without closing channel | New commitment structure mid-life, signer sees new funding outpoint |

---

## Adversarial / Failure Scenarios (1000+)

### Protocol-level failures (1001–1099)

Scenarios where the Lightning protocol itself handles failure — no attacker, just unhappy paths.

| # | Name | Description | VLS relevance |
|---|------|-------------|---------------|
| 1001 | HTLC timeout (off-chain) | Payment fails, removed via `update_fail_htlc` | Signer allows revocation without preimage — normal failure path |
| 1002 | HTLC timeout (on-chain) | HTLC expires, force close needed | Signer must sign HTLC-timeout 2nd-stage tx |
| 1003 | Unilateral close (force close) | One side broadcasts their commitment tx | Signer signs `SignLocalCommitmentTx2`, then `SignDelayedPaymentToUs` after CSV. Also HTLC sweep txs. |
| 1004 | Breach — revoked state broadcast | Counterparty publishes old commitment | Signer signs `SignPenaltyToUs` using revealed revocation secret |
| 1005 | Invalid sig from counterparty | Counterparty sends bad `commitment_signed` | Signer rejects at `ValidateCommitmentTx` — channel should not advance |
| 1007 | Reconnect mid-update | Disconnect during commitment exchange | `channel_reestablish` to resync — signer handles replayed/retried calls |

### Compromised node — on-chain theft (1100–1199)

Attacker controls the node, tries to steal funds via on-chain transactions.

| # | Name | Description | VLS relevance | Primary policy |
|---|------|-------------|---------------|----------------|
| 1101 | Fabricated output in funding tx | Attacker adds extra output to funding tx sending to their address | Signer rejects unknown outputs | `policy-onchain-no-unknown-outputs` |
| 1102 | Fee siphoning (on-chain) | Attacker inflates funding tx fee, recovers via colluding miner | Signer enforces fee bounds | `policy-onchain-fee-range` |
| 1103 | Non-SegWit input malleation | Attacker uses legacy input in funding tx — txid changes after broadcast, all commitments invalidated, funds locked permanently | Signer requires SegWit inputs | `policy-onchain-funding-non-malleable` |
| 1104 | Redirect mutual close proceeds | Attacker substitutes shutdown script with their own address | Signer checks destination allowlist | `policy-mutual-destination-allowlisted` |
| 1105 | Redirect force-close sweep | Attacker sends `SignDelayedPaymentToUs` with attacker's address as output | Signer checks destination allowlist | `policy-sweep-destination-allowlisted` |
| 1106 | Bad locktime on sweep | Attacker sets locktime far in future, delaying fund recovery | Signer checks locktime against chain height | `policy-sweep-locktime` |

### Compromised node — commitment/HTLC theft (1200–1299)

Attacker controls the node, tries to steal funds via commitment or HTLC manipulation.

| # | Name | Description | VLS relevance | Primary policy |
|---|------|-------------|---------------|----------------|
| 1201 | Fee siphoning (commitment) | Attacker accepts absurd feerate from counterparty, excess goes to colluding miner | Signer enforces fee bounds on commitments | `policy-commitment-fee-range` |
| 1202 | Sign revoked commitment | Attacker requests signature on old (revoked) commitment; if broadcast, counterparty takes all via penalty | Signer tracks revocation state | `policy-commitment-holder-not-revoked` |
| 1203 | HTLC routing balance attack | Attacker offers outgoing HTLC with no matching inbound — drains channel to colluding node | Signer checks balance of offered vs received HTLCs | `policy-commitment-htlc-routing-balance` |
| 1204 | Rapid fund drainage | Attacker pays colluding nodes as fast as possible before detection | Signer rate-limits outgoing payments | `policy-commitment-payment-velocity` |
| 1205 | Overpay invoice | Attacker preapproves small invoice then sends much larger HTLC | Signer checks HTLC amount against approved invoice | `policy-commitment-payment-invoiced` |
| 1206 | Fee siphoning (mutual close) | Attacker inflates cooperative close fee | Signer enforces fee bounds on close | `policy-mutual-fee-range` |
| 1207 | Close with pending HTLCs | Attacker tries to close while HTLCs still in-flight, forfeiting those funds | Signer refuses close with pending HTLCs | `policy-mutual-no-pending-htlcs` |

---

## Additional Signer Calls Exercised

These calls appear in adversarial/failure scenarios but not in the happy-path scenario 01:

| Call | Used in | Purpose |
|------|---------|---------|
| `PreapproveInvoice` | 1203, 1204, 1205 | Validate invoice before allowing payment |
| `SignLocalCommitmentTx2` | 1003 | Sign our own commitment for force-close broadcast |
| `SignDelayedPaymentToUs` | 1003, 1105, 1106 | Sign sweep of to_local output after CSV delay |
| `SignPenaltyToUs` | 1004 | Sign justice tx claiming all funds from revoked commitment |
| `SignHtlcTx` | 1002, 1003 | Sign 2nd-stage HTLC-timeout or HTLC-success tx |
| `ForgetChannel` | 01 (close phase) | Clean up signer state after channel fully closed |
| `AddBlock` | 1003, 1106 | Feed chain data so signer can validate timelocks |

---

## Priorities

**Next up:** Scenario 02 (routing) — this is the actual VLS deployment model. A routing node's signer must handle two channels simultaneously, validating that the forwarded HTLC matches the incoming one (amount, timelock delta).

**After that:** Scenario 1003 (force close + sweep) — exercises `SignLocalCommitmentTx2`, `SignDelayedPaymentToUs`, `AddBlock`, and the full chain-validated sweep flow.

**Then:** Scenario 1004 (breach/penalty) — exercises `SignPenaltyToUs` and validates the full revocation secret → penalty tx path.

**Adversarial scenarios** (1100+, 1200+) are best documented after the happy-path flows they attack are solid.
