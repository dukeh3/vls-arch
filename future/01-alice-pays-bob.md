# Scenario 01 — Alice Pays Bob (Future — v2 proxy protocol)

Same scenario as [`../current/01-alice-pays-bob.md`](../current/01-alice-pays-bob.md), but introduces **two proxy layers** between each node and its signer, with a **semantic proxy-to-proxy protocol** that compresses VLS calls on the expensive cross-trust-boundary link.

## What changed from v1

| | v1 | v2 |
|---|---|---|
| Proxies per side | 1 (inline passthrough) | 2 — node-proxy + signer-proxy |
| Proxy↔proxy traffic | n/a | Named messages — one RTT per protocol event |
| Node interface | unchanged from `current/` | unchanged from `current/` |
| Signer interface | unchanged from `current/` | unchanged from `current/` |
| Goal | Establish the seam | **Compress the number of round-trips on the cross-trust-boundary link** |

The motivation: the node-proxy → signer-proxy link is the expensive one — it crosses a trust boundary (potentially the internal relay, vsock-to-host-to-relay, or a network hop). The signer-proxy → signer link is local and cheap (vsock or in-process). So we compress where it matters most: between the two proxies.

The node still issues individual VLS calls. The signer still receives individual VLS calls. The compression happens entirely between the proxies.

## Architecture

```
   Alice  ────→  node-proxy    ════→    signer-proxy  ────→  Alice Signer
                 (buffers VLS   named    (unpacks,
                  calls,        proxy     dispatches to
                  translates)   msgs)     signer)

   Bob    ────→  node-proxy    ════→    signer-proxy  ────→  Bob Signer
```

- **Node-proxy** — receives individual VLS calls from the node, translates them into named proxy messages, handles deferral (e.g., returns dummy values for calls whose replies are discarded by LDK).
- **Signer-proxy** — receives proxy messages, unpacks them into VLS calls, dispatches to the signer sequentially, collects responses, returns the reply.
- **Proxy-to-proxy protocol** — named messages with semantic meaning, not generic batches.

## Proxy-to-proxy messages

### Channel open (section 01)

| Message | Purpose | RTTs |
|---------|---------|------|
| [`vls_create_channel`](01-alice-pays-bob-01-open-channel-A01.md) | Create channel + get basepoints + per-commitment point | 1 |
| [`vls_funding_created`](01-alice-pays-bob-01-open-channel-A02.md) | Setup channel + sign remote commitment | 1 |
| [`vls_funding_signed`](01-alice-pays-bob-01-open-channel-B02.md) | Setup + validate own commitment + sign remote commitment | 1 |
| [`vls_sign_funding`](01-alice-pays-bob-01-open-channel-A03.md) | Validate own commitment + sign funding transaction | 1 |
| [`vls_channel_ready`](01-alice-pays-bob-01-open-channel-A04.md) | Confirm outpoint + lock + get next per-commitment point | 1 |

### Commitment updates (section 02)

Each state change (HTLC add, settle, etc.) requires **3 independent RTTs**:

| Message | Purpose | RTTs |
|---------|---------|------|
| [`vls_commitment_signed`](01-alice-pays-bob-02-htlc-add-A01.md) | Sign counterparty's new commitment | 1 |
| [`vls_revoke_commitment`](01-alice-pays-bob-02-htlc-add-B01.md) | Validate own new commitment + revoke old → `revoke_and_ack` | 1 |
| [`vls_validate_revocation`](01-alice-pays-bob-02-htlc-add-A02.md) | Validate counterparty's revocation secret | 1 |

These messages are **independent** — no ordering assumptions, no bundling. Each travels separately and can be processed in any order. Both directions (Alice→Bob and Bob→Alice) run the same three messages in parallel.

### Cooperative close (section 04)

| Message | Purpose | RTTs |
|---------|---------|------|
| [`vls_closing_signed`](01-alice-pays-bob-04-close-A01.md) | Sign cooperative close transaction | 1 |

## Scenario sections

| Section | Files | Description |
|---------|-------|-------------|
| 01 — Channel open | [A01](01-alice-pays-bob-01-open-channel-A01.md), [B01](01-alice-pays-bob-01-open-channel-B01.md), [A02](01-alice-pays-bob-01-open-channel-A02.md), [B02](01-alice-pays-bob-01-open-channel-B02.md), [A03](01-alice-pays-bob-01-open-channel-A03.md), [A04](01-alice-pays-bob-01-open-channel-A04.md), [B03](01-alice-pays-bob-01-open-channel-B03.md) | Alice funds 1.0 BTC channel |
| 02 — HTLC add | [A01](01-alice-pays-bob-02-htlc-add-A01.md), [B01](01-alice-pays-bob-02-htlc-add-B01.md), [A02](01-alice-pays-bob-02-htlc-add-A02.md) | Alice sends 0.2 BTC to Bob via HTLC |
| 03 — HTLC settle | (note in [A02](01-alice-pays-bob-02-htlc-add-A02.md#htlc-settlement-section-03)) | Same mechanism as 02, different balances |
| 04 — Cooperative close | [A01](01-alice-pays-bob-04-close-A01.md), [B01](01-alice-pays-bob-04-close-B01.md) | Both sides sign and broadcast close tx |

## Compression score

End-to-end count of cross-proxy round-trips, current vs v2:

| Phase | Current (per side) | v2 (per side) |
|-------|-------------------|---------------|
| Channel open | 7–10 calls | **5 RTTs** |
| Each commitment update | 3 calls | **3 RTTs** (no compression — already minimal) |
| Cooperative close | 1 call | **1 RTT** |
| **Full scenario** | ~19 calls (Alice) | **12 RTTs** |

The main compression win is in channel open, where multiple VLS calls that are independent (NewChannel + GetChannelBasepoints + GetPerCommitmentPoint) get folded into a single named message (`vls_create_channel`). Commitment updates are already 1 call per protocol event — the proxy adds semantic naming and deferral (combining ValidateCommitmentTx2 + RevokeCommitmentTx into one `vls_revoke_commitment` RTT) but the RTT count per state change remains 3.

## Why messages end where they do

The proxy messages are bounded by **protocol moments** — points where a Lightning wire message must be sent or received:

- **`vls_commitment_signed`** — fires when the node needs to send `commitment_signed` to the peer. One call, one reply needed immediately.
- **`vls_revoke_commitment`** — fires when the node receives `commitment_signed` from the peer. Combines validate + revoke because LDK discards ValidateCommitmentTx2's reply, so the node-proxy can defer it and flush both when RevokeCommitmentTx arrives.
- **`vls_validate_revocation`** — fires when the node receives `revoke_and_ack` from the peer. Single call, empty reply.
- **`vls_closing_signed`** — fires when the node needs to send `closing_signed` to the peer.

Channel open messages follow the same pattern — each fires at a protocol boundary where a Lightning wire message (open_channel, accept_channel, funding_created, funding_signed, channel_ready) is sent or received.

## Roadmap context

| Version | What the proxy pair adds |
|---------|--------------------------|
| **v1** | Single inline passthrough proxy (the structural seam) |
| **v2** | **Named proxy messages with call compression — this document** |
| **v3** | Signer-proxy ingests NWC/NNC events for `kind:30078` UsageProfile enforcement |
| **v4** | Signer-proxy ingests TXOO chain attestations alongside proxy messages |
| **v5** | Proxy-to-proxy link runs over the internal Nostr relay — see [Nostr Signer Connect](nostr-signer-connect.md) |

Each step extends one of the two proxies; the node, the signer, and the BOLT-level VLS protocol stay stable.

## Source References

Wire message definitions — `vls-protocol/src/msgs.rs`:

| Call | Message struct | Line |
|------|---------------|------|
| NewChannel | `NewChannel` | [msgs.rs:487](../validating-lightning-signer/vls-protocol/src/msgs.rs#L487) |
| GetChannelBasepoints | `GetChannelBasepoints` | [msgs.rs:231](../validating-lightning-signer/vls-protocol/src/msgs.rs#L231) |
| GetPerCommitmentPoint | `GetPerCommitmentPoint` | [msgs.rs:337](../validating-lightning-signer/vls-protocol/src/msgs.rs#L337) |
| SetupChannel | `SetupChannel` | [msgs.rs:500](../validating-lightning-signer/vls-protocol/src/msgs.rs#L500) |
| SignRemoteCommitmentTx2 | `SignRemoteCommitmentTx2` | [msgs.rs:353](../validating-lightning-signer/vls-protocol/src/msgs.rs#L353) |
| ValidateCommitmentTx2 | `ValidateCommitmentTx2` | [msgs.rs:566](../validating-lightning-signer/vls-protocol/src/msgs.rs#L566) |
| RevokeCommitmentTx | `RevokeCommitmentTx` | [msgs.rs:644](../validating-lightning-signer/vls-protocol/src/msgs.rs#L644) |
| ValidateRevocation | `ValidateRevocation` | [msgs.rs:587](../validating-lightning-signer/vls-protocol/src/msgs.rs#L587) |
| CheckOutpoint | `CheckOutpoint` | [msgs.rs:524](../validating-lightning-signer/vls-protocol/src/msgs.rs#L524) |
| LockOutpoint | `LockOutpoint` | [msgs.rs:600](../validating-lightning-signer/vls-protocol/src/msgs.rs#L600) |
| SignMutualCloseTx2 | `SignMutualCloseTx2` | [msgs.rs:931](../validating-lightning-signer/vls-protocol/src/msgs.rs#L931) |

Handler implementations — `vls-protocol-signer/src/handler.rs`:

| Handler | Line |
|---------|------|
| SignRemoteCommitmentTx2 | [handler.rs:1308](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1308) |
| SignMutualCloseTx2 | [handler.rs:1402](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1402) |
| ValidateCommitmentTx2 | [handler.rs:1472](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1472) |
| RevokeCommitmentTx | [handler.rs:1526](../validating-lightning-signer/vls-protocol-signer/src/handler.rs#L1526) |

Parameters verified against VLS source at commit `75e3a46b`. Diagrams assume **protocol version 6** (current default, `DEFAULT_MAX_PROTOCOL_VERSION`) — separate `RevokeCommitmentTx`, `GetPerCommitmentPoint` returns point only.

For notation (`Af`, `Ar`, `ap0`, `rp(...)`, etc.) see [Key Derivation](../reference/lightning-key-derivation.md#notation-legend).
