# Questions and Challenges

Open questions and known challenges with the current VLS signer protocol.

---

## 1. Multiple signer calls per state change

A single Lightning state transition requires multiple separate calls to the signer. For example, during commitment 0 (channel open), Alice's node makes 10 separate signer calls:

1. NewChannel
2. GetChannelBasepoints
3. GetPerCommitmentPoint(0)
4. SetupChannel
5. SignRemoteCommitmentTx
6. ValidateCommitmentTx
7. SignWithdrawal
8. CheckOutpoint
9. LockOutpoint
10. GetPerCommitmentPoint(1)

Each call is a round-trip to the signer. In deployments where the signer is remote (e.g., on a separate device or across a network), this adds latency to every state transition. A single HTLC add-and-settle cycle (commitments 1 and 2) requires 7 signer calls per side, per commitment — 14 calls total for what is conceptually one payment.

**Questions:**
- Which of these calls could be combined without weakening the signer's ability to validate?
- Are there calls that exist only because of implementation history rather than security requirements?
- What is the minimum set of signer interactions needed per state change?

---

## 2. Implicit dependencies between signer calls

The signer calls have ordering dependencies that are not expressed in the API. The caller (the node) must know the correct sequence — the signer does not enforce it.

Examples:

- **SetupChannel must come after accept_channel** — it requires both sides' parameters, but nothing in the API prevents calling it earlier (it would fail or produce wrong results).
- **ValidateCommitmentTx must come before RevokeCommitmentTx** — you should validate the new commitment before revoking the old one. If the node revokes first and the new commitment turns out to be invalid, the old state is already gone. The signer doesn't enforce this ordering.
- **SignRemoteCommitmentTx before sending commitment_signed** — obvious, but the signer doesn't know whether the signature was actually sent to the peer.
- **CheckOutpoint + LockOutpoint before GetPerCommitmentPoint(1)** — the channel should be locked to the confirmed funding outpoint before advancing to the next state. But the signer allows these in any order.

The signer tracks some state (commitment numbers, revocation secrets) but largely trusts the node to call methods in the right order. This means a buggy node (not even a compromised one) could put the signer into an inconsistent state by calling methods out of sequence.

**Questions:**
- Should the signer enforce call ordering via an explicit state machine?
- Would a state machine in the signer catch bugs earlier without adding unwanted rigidity?
- Are there cases where the "wrong" order is actually valid (e.g., concurrent channels, reconnect flows)?

---

## 3. Call names hide state-changing side effects

Several signer calls do more than their names suggest. They appear to be queries or validations, but they also silently update the signer's internal state. The call ordering in challenge #2 matters precisely because of these hidden side effects.

Examples:

| Call | Name suggests | Actually also does |
|------|--------------|-------------------|
| `ValidateCommitmentTx` | Check a signature | Stores the new commitment as current state. Returns `next_per_commitment_point`. In protocol < 5: also revokes the previous commitment and returns `old_commitment_secret`. In protocol >= 5: calls `activate_initial_commitment()` for commitment 0. ([handler.rs:1415](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1415)) |
| `RevokeCommitmentTx` | Produce a revocation secret | Advances the signer's commitment counter. Returns `old_commitment_secret` + `next_per_commitment_point`. Only exists in protocol >= 5 — before that, revocation was hidden inside ValidateCommitmentTx. ([handler.rs:1526](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1526)) |
| `ValidateRevocation` | Check a revocation secret | Stores the counterparty's secret. The signer now has the material to sign a penalty tx if the counterparty broadcasts the revoked state. ([handler.rs:1556](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1556)) |
| `LockOutpoint` | Lock the funding UTXO | Transitions the channel from "pending" to "active" in the signer's state. (Currently a placeholder — [handler.rs:1258](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1258)) |
| `GetPerCommitmentPoint(N)` | Return a public point | In protocol < 6: also returns the secret for commitment N-2. ([handler.rs:1171](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1171)) |

This creates a mismatch between the API's apparent semantics (validate, get, check) and its actual semantics (validate-and-advance, get-and-prepare, check-and-transition). The caller must understand the hidden state transitions to use the API correctly, but nothing in the API surface communicates them.

The protocol version history shows these side effects being gradually untangled:
- **Protocol 5** (`PROTOCOL_VERSION_REVOKE`): Revocation split out from `ValidateCommitmentTx` into explicit `RevokeCommitmentTx` ([msgs.rs:39](../validating-lightning-signer/vls-protocol/src/msgs.rs#L39))
- **Protocol 6** (`PROTOCOL_VERSION_NO_SECRET`): `GetPerCommitmentPoint` no longer returns old secrets ([msgs.rs:40](../validating-lightning-signer/vls-protocol/src/msgs.rs#L40))

This trend suggests the codebase is moving toward cleaner separation of concerns, but the return of `next_per_commitment_point` from both `ValidateCommitmentTx` and `RevokeCommitmentTx` is still a hidden side effect — a "validate" or "revoke" call doubling as a "get next point" call.

**Questions:**
- Should calls that mutate state be named to reflect it (e.g., `AcceptCommitmentTx` instead of `ValidateCommitmentTx`)?
- Should pure validation be separated from state advancement (validate without committing, then explicitly commit)?
- Would explicit state transitions make the protocol easier to reason about for both implementors and auditors?

---

## 4. Redundant parameters across calls

The signer doesn't track Lightning `update_*` messages — it has no knowledge of HTLCs being added or settled until the node describes the full commitment state in each call. This means every Sign and Validate call re-sends information the signer has already seen or could derive.

### SignRemoteCommitmentTx and ValidateCommitmentTx share most parameters

These two calls describe the **same commitment state** from opposite perspectives — one signs the remote's version, the other validates the local's version. Their wire message structs are nearly identical:

| Field | SignRemoteCommitmentTx | ValidateCommitmentTx |
|-------|:---------------------:|:--------------------:|
| `tx` | ✓ | ✓ |
| `psbt` | ✓ | ✓ |
| `commitment_number` | ✓ | ✓ |
| `feerate` | ✓ | ✓ |
| `htlcs` | ✓ | ✓ |
| `remote_funding_key` | ✓ | — |
| `remote_per_commitment_point` | ✓ | — |
| `option_static_remotekey` | ✓ | — |
| `signature` | — | ✓ |
| `htlc_signatures` | — | ✓ |

Source: [SignRemoteCommitmentTx — msgs.rs:353](../validating-lightning-signer/vls-protocol/src/msgs.rs#L353), [ValidateCommitmentTx — msgs.rs:566](../validating-lightning-signer/vls-protocol/src/msgs.rs#L566)

5 of 8 fields in `SignRemoteCommitmentTx` also appear in `ValidateCommitmentTx`. The only differences are call-specific: Sign needs the remote's keys to produce a signature, Validate receives a signature to check. The commitment state itself (`tx`, `psbt`, `commitment_number`, `feerate`, `htlcs`) is sent identically both times.

The `_phase2` variants make the overlap even more explicit — they drop the full `tx` and `psbt` in favour of explicit balance values, but the shared fields remain:

| Field | SignRemoteCommitmentTx2 | ValidateCommitmentTx2 |
|-------|:----------------------:|:---------------------:|
| `commitment_number` | ✓ | ✓ |
| `feerate` | ✓ | ✓ |
| `to_local_value_sat` | ✓ | ✓ |
| `to_remote_value_sat` | ✓ | ✓ |
| `htlcs` | ✓ | ✓ |
| `remote_per_commitment_point` | ✓ | — |
| `signature` | — | ✓ |
| `htlc_signatures` | — | ✓ |

Source: [SignRemoteCommitmentTx2 — msgs.rs:910](../validating-lightning-signer/vls-protocol/src/msgs.rs#L910), [ValidateCommitmentTx2 — msgs.rs:943](../validating-lightning-signer/vls-protocol/src/msgs.rs#L943)

### The core problem: signing and validation can diverge

Because the commitment state is described independently in each call, the signer has no guarantee that `SignRemoteCommitmentTx` and `ValidateCommitmentTx` received the same state. A buggy or compromised node could:

1. Call `SignRemoteCommitmentTx` with htlcs=[HTLC_A] — the signer signs Bob's commitment containing HTLC_A
2. Call `ValidateCommitmentTx` with htlcs=[HTLC_B] — the signer validates Alice's commitment containing HTLC_B

The signer processes both calls independently. It signed a remote commitment with one set of HTLCs and accepted a local commitment with a different set. Nothing in the protocol ties these two calls together or verifies they describe the same state transition.

In the normal Lightning protocol, Alice's and Bob's commitments for the same state are mirror images — they must contain the same HTLCs, the same feerate, and complementary balances. But the signer has no way to enforce this because it sees each call in isolation. The node is trusted to provide consistent data across calls.

This is not just redundancy — it's a gap in the signer's validation. The signer validates each commitment individually (correct signatures, sane fees, balanced amounts) but cannot verify that the two commitments it processed actually correspond to the same channel state.

Additionally:
- The `to_local`/`to_remote` values between the two calls should be mirror images (local's `to_remote` = remote's `to_local`), but the signer doesn't cross-check this
- `channel_value` was already provided in `SetupChannel`, so balances are derivable from the HTLCs and fees — yet both calls re-specify them independently
- The signer cannot detect whether an HTLC was legitimately added via the protocol or fabricated by a compromised node — it only sees the final state, not how it got there

**Questions:**
- Should the signer cross-check that Sign and Validate for the same commitment number describe the same state?
- Could Sign and Validate share a single "describe this commitment" call, with separate "now sign" / "now validate" follow-ups — eliminating the possibility of divergence?
- If the signer tracked updates (like a lightweight protocol state machine), could it eliminate redundant parameters entirely and derive the expected state itself?
- Would this also address challenge #2 (implicit dependencies) by giving the signer enough context to enforce ordering?
- Is the redundancy actually a feature? (Cross-checking redundant data catches bugs in the node implementation — but only if the signer actually cross-checks, which it currently does not)

---

## 5. Protocol versioning changes call semantics

The VLS signer protocol is versioned, and the version is negotiated at init time between the node and signer (`min(signer_max, node_max)`). Different versions change which calls exist and what they return — the same call name can have different semantics depending on the negotiated version.

Known version gates ([msgs.rs:39-44](../validating-lightning-signer/vls-protocol/src/msgs.rs#L39)):

| Version | Constant | Change |
|---------|----------|--------|
| 2 | `MIN_PROTOCOL_VERSION` | Minimum supported version |
| < 5 | `PROTOCOL_VERSION_REVOKE` | `ValidateCommitmentTx` validates AND revokes in one call. `RevokeCommitmentTx` does not exist. |
| >= 5 | | Revocation split into explicit `RevokeCommitmentTx`. Advertised as a capability in `HsmdInitReplyV4`. |
| < 6 | `PROTOCOL_VERSION_NO_SECRET` | `GetPerCommitmentPoint(N)` returns point AND secret for N-2. |
| >= 6 | | `GetPerCommitmentPoint` returns only the point. This is the current default (`DEFAULT_MAX_PROTOCOL_VERSION`). |

The version negotiation happens during `HsmdInitReplyV4` ([handler.rs:599](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L599)). The signer advertises `RevokeCommitmentTx` as a capability only when mutual version >= 5 ([handler.rs:627](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L627)).

This compounds the challenges above:
- Challenge #3 (hidden side effects) depends on protocol version — the same call does different things in v4 vs v6
- Documentation must specify which protocol version it describes
- The node and signer must agree on version, adding another coordination point
- Old call semantics must be supported indefinitely for backwards compatibility, increasing complexity

**Questions:**
- Should there be a clean break where old protocol versions are dropped?
- Can the `_phase2` method variants (which take explicit values instead of full transactions) become the standard API, replacing the version-gated behavior?
- How does protocol versioning interact with multi-signer setups (epic 2) — do all signers need the same version?
