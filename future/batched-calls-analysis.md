# Batched calls — code verification & combine proposals

Companion to [`01-alice-pays-bob.md`](./01-alice-pays-bob.md) (v2 two-proxy batching). For each batch in that scenario, this file:

1. **Restates** the batch from the diagram
2. **Verifies** wire-struct fields and handler behavior against the VLS source at commit `75e3a46b`
3. **Lists** the variables flowing in and out
4. **Proposes** a single combined message that could replace the batch in a future protocol version

The combined messages are design proposals, not implementations. They motivate a hypothetical **v3 wire protocol** in which common multi-call patterns become single round-trips at the signer level (not just at the proxy level, the way v2 batching does).

---

## Batch A0-1 — Initial setup (Alice, funder)

### The batch

| # | Call | Message ID | Inputs (from node) | Outputs (to node) |
|---|------|-----------|--------------------|--------------------|
| 1 | `NewChannel` | 30 | `peer_id`, `dbid` | (empty `NewChannelReply`) |
| 2 | `GetChannelBasepoints` | 10 | `node_id`, `dbid` | `Basepoints` + `funding` |
| 3 | `GetPerCommitmentPoint` | 18 | `commitment_number=0` | `point`, `secret` |

### Verified wire structs

From [`vls-protocol/src/msgs.rs`](../validating-lightning-signer/vls-protocol/src/msgs.rs):

```rust
// L487
pub struct NewChannel {
    pub peer_id: PubKey,   // 33 bytes — Bob's node pubkey
    pub dbid: u64,         // channel database id, node-assigned
}
pub struct NewChannelReply {}  // empty

// L231
pub struct GetChannelBasepoints {
    pub node_id: PubKey,   // 33 bytes — Alice's own node pubkey
    pub dbid: u64,
}
pub struct GetChannelBasepointsReply {
    pub basepoints: Basepoints,
    pub funding: PubKey,   // Af
}

// vls-protocol/src/model.rs:155
pub struct Basepoints {
    pub revocation: PubKey,        // Ar
    pub payment: PubKey,           // Ap
    pub htlc: PubKey,              // Ah
    pub delayed_payment: PubKey,   // Ad
}

// L337
pub struct GetPerCommitmentPoint {
    pub commitment_number: u64,
}
pub struct GetPerCommitmentPointReply {
    pub point: PubKey,                   // ap0
    pub secret: Option<DisclosedSecret>, // None at protocol v6+
}
```

**Wire-format field order for `Basepoints`** is `(revocation, payment, htlc, delayed_payment)` — i.e. `(Ar, Ap, Ah, Ad)`. The scenario diagram lists them as `(Ar, Ap, Ad, Ah)` for presentation, which is a display reordering only.

### Verified handler behavior

From [`vls-protocol-signer/src/handler.rs`](../validating-lightning-signer/vls-protocol-signer/src/handler.rs):

| Call | Handler kind | Line | Notes |
|------|--------------|------|-------|
| `NewChannel` | `RootHandler::do_handle` | 750 | calls `self.node.new_channel(dbid, peer_id, &node)` |
| `GetChannelBasepoints` | `RootHandler::do_handle` | 759 | derives `channel_id = Self::channel_id(node_id, dbid)`, then `with_channel_base(...).get_channel_basepoints()` |
| `GetPerCommitmentPoint` | `ChannelHandler::do_handle` | 1228 | uses already-bound `self.channel_id`; `secret` is `None` at protocol ≥ 6 (`PROTOCOL_VERSION_NO_SECRET`) |

> **Doc fix applied** in `current/01-alice-pays-bob.md`: handler lines were listed as 748 and 1171 respectively; the correct lines are **750** and **1228**.

### Variables Alice's node must already have

| Variable | Type | Source |
|----------|------|--------|
| `peer_id` (Bob) | 33-byte pubkey | Peer discovery (gossip, manual) |
| `node_id` (Alice's own) | 33-byte pubkey | Local node key |
| `dbid` | u64 | Local counter, unique per channel |
| `commitment_number` | u64 | Constant `0` for initial commit |

### Variables Alice learns from the batch

| Variable | From call | Used next in |
|----------|-----------|--------------|
| `Ar` revocation basepoint | 2 | `open_channel` message to Bob |
| `Ap` payment basepoint | 2 | `open_channel` |
| `Ah` htlc basepoint | 2 | `open_channel` |
| `Ad` delayed-payment basepoint | 2 | `open_channel` |
| `Af` funding pubkey | 2 | `open_channel`; 2-of-2 multisig output |
| `ap0` per-commitment point[0] | 3 | `open_channel`; defines Bob's first commit outputs |

After the batch returns, Alice immediately sends `open_channel(Af, Ar, Ap, Ad, Ah, ap0, 1.0 BTC, to_self_delay=10)`. The batch's job is to produce exactly the six pubkeys this message needs.

### State dependencies within the batch

- **NewChannel → GetChannelBasepoints**: the basepoint handler does `with_channel_base(&channel_id)`, which requires the channel to exist. NewChannel must complete first.
- **NewChannel → GetPerCommitmentPoint**: same reason — the per-commitment-point handler also uses `with_channel_base`.
- **GetChannelBasepoints ⊥ GetPerCommitmentPoint**: independent of each other; could run in either order.

The signer-proxy's sequential dispatch handles the ordering naturally. The proxy-to-proxy `BatchCall` is one round-trip; the signer-side dispatch is three calls in order.

### Observation — `node_id` in `GetChannelBasepoints` is redundant

The `node_id` field in `GetChannelBasepoints` is always the signer's own node identity. It's used by `Self::channel_id(&node_id, dbid)` to derive the internal channel id. In single-node VLS deployments this is fully redundant — the signer already knows its own node id. This is a CLN-ism (hsmd was multi-tenant in some early designs) carried into the VLS protocol. A combined message can drop this field.

---

## Proposed combined message: `InitChannel`

A single message that performs the work of all three Batch A0-1 calls in one signer round-trip.

### Wire format

```rust
#[derive(SerBolt, Debug, Encodable, Decodable)]
#[message_id(50)]   // hypothetical — pick a free id
pub struct InitChannel {
    pub peer_id: PubKey,   // counterparty (Bob)
    pub dbid: u64,         // node-assigned channel id
    // commitment_number is implicit (= 0 for "init")
    // node_id is implicit (signer knows its own identity)
}

#[derive(SerBolt, Debug, Encodable, Decodable)]
#[message_id(150)]
pub struct InitChannelReply {
    pub basepoints: Basepoints,           // Ar, Ap, Ah, Ad
    pub funding: PubKey,                  // Af
    pub initial_per_commitment_point: PubKey,   // ap0
}
```

### What's elided versus the three-call sequence

- `node_id` from `GetChannelBasepoints` — always the signer's own identity, derivable on the signer side
- `commitment_number` from `GetPerCommitmentPoint` — always 0 for a fresh channel
- Three separate `Reply` envelopes fused into one
- The empty `NewChannelReply` — implicit success on receiving a non-error `InitChannelReply`

The proposed message is **2 fields in, 6 pubkeys out** versus the current **4 fields total in across three requests, 7 fields out across three responses**.

### Handler sketch

```rust
Message::InitChannel(m) => {
    // 1. NewChannel-equivalent
    self.node.new_channel(m.dbid, &m.peer_id.0, &self.node)?;

    // 2. GetChannelBasepoints-equivalent — node_id is implicit
    let our_node_id = PubKey(self.node.get_id().serialize());
    let channel_id = Self::channel_id(&our_node_id, m.dbid);
    let bps = self.node.with_channel_base(&channel_id, |base|
        Ok(base.get_channel_basepoints()))?;

    // 3. GetPerCommitmentPoint-equivalent — commitment_number=0 implicit
    let point = self.node.with_channel_base(&channel_id, |base|
        base.get_per_commitment_point(0))?;

    Ok(Box::new(msgs::InitChannelReply {
        basepoints: Basepoints {
            revocation:      PubKey(bps.revocation_basepoint.0.serialize()),
            payment:         PubKey(bps.payment_point.serialize()),
            htlc:            PubKey(bps.htlc_basepoint.0.serialize()),
            delayed_payment: PubKey(bps.delayed_payment_basepoint.0.serialize()),
        },
        funding: PubKey(bps.funding_pubkey.serialize()),
        initial_per_commitment_point: PubKey(point.serialize()),
    }))
}
```

The implementation reuses the same internal node functions; only the protocol surface changes.

### Trade-offs

| Concern | v2 batched (today) | Proposed `InitChannel` |
|---------|--------------------|------------------------|
| Cross-proxy round-trips | 1 (BatchCall) | 1 |
| Signer handler invocations | 3 sequential | 1 atomic |
| Wire-protocol surface | 3 existing messages | +1 new message |
| Failure atomicity | Partial: if call 2 errors after call 1 succeeded, channel state exists with no basepoints retrieved | All-or-nothing |
| Reuse outside this pattern | Each call has other uses (e.g. `GetPerCommitmentPoint(N)` for any N) | Single-purpose (init only) |
| Backward compatibility | n/a | Requires protocol version bump; old signers reject unknown msg-id |
| Proxy complexity | Proxy is generic — batches anything | Proxy is generic — passes through anything |
| Signer code | Unchanged | New handler + handler test |

### Two architectural paths to "one call"

The proposal above is **path B**. Path A is also viable.

**Path A — Proxy-side fusion.** The v2 node-proxy detects the three-call pattern and synthesizes the equivalent signer-side work locally, never sending three separate messages. Pros: no new signer protocol. Cons: tightly couples the proxy to the signer's channel-id derivation, state model, and error semantics — every new signer version risks breaking the proxy's local reimplementation.

**Path B — New signer operation (proposed above).** The signer gains an `InitChannel` handler. The node (or its proxy) issues `InitChannel` instead of three calls. Pros: single source of truth in the signer; proxy stays generic. Cons: protocol surface grows; needs version negotiation in `HsmdInit`.

Path B is cleaner. The v2 proxy continues to do generic batching of legacy messages (for protocol fallback when talking to a signer that doesn't speak v3); when both sides advertise v3, the node-proxy issues `InitChannel` directly with no batching needed.

### Compression metrics (Batch A0-1 only)

| Metric | Current (3 calls) | v2 batched | InitChannel (proposed) |
|--------|-------------------|------------|------------------------|
| Cross-proxy round-trips | 3 | 1 | 1 |
| Signer handler invocations | 3 | 3 | 1 |
| Wire bytes — request (approx) | 33 + 33 + 33 + 8 + 8 + 8 = 123 | 123 + small batch header | 33 + 8 = 41 |
| Wire bytes — reply (approx) | 0 + 165 + 33 = 198 | 198 + small batch header | 33×5 + 33 = 198 |

The big wins are on the **request side** (~3× smaller) and **signer-side dispatch** (3→1). The reply is the same size because the same six pubkeys are returned.

---

## Next batches (placeholders)

Subsequent batches to be analyzed in this file as we work through them:

- **Batch B0-1** — Bob's symmetric initial response (NewChannel + GetChannelBasepoints + GetPerCommitmentPoint(0))
- **Batch A0-2** — SetupChannel + SignRemoteCommitmentTx
- **Batch B0-2** — SetupChannel + ValidateCommitmentTx + SignRemoteCommitmentTx
- **Batch A0-3** — ValidateCommitmentTx + SignWithdrawal
- **Batch A0-4 / B0-3** — CheckOutpoint + LockOutpoint + GetPerCommitmentPoint(1)
- **Batch A1-1 / B1-1** — Commitment 1 update-and-sign
- **Batch A1-2 / B1-2** — Commitment 1 revocation processing
- **Batch A2-1 / B2-1** — Commitment 2 settle-and-sign
- **Batch A2-2 / B2-2** — Commitment 2 revocation processing
- **Close batches** — single-call SignMutualCloseTx on each side
