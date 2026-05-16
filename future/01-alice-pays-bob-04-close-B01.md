# Cooperative Close B01 — Bob Signs and Broadcasts

Part of [Scenario 01 — Alice Pays Bob](01-alice-pays-bob.md). Follows [close-A01](01-alice-pays-bob-04-close-A01.md).

**Scope:** From Bob receiving Alice's `closing_signed` until the closing transaction is broadcast.

---

## Lightning Context

Bob receives `closing_signed(fee, sig_Af)` from Alice. He now:
1. Validates the fee is acceptable
2. Verifies Alice's signature (`sig_Af`) on the closing transaction — done by the node, not the signer
3. Signs the closing transaction with his funding key (`Bf`)
4. Sends `closing_signed(fee, sig_Bf)` to Alice
5. Broadcasts the closing transaction (has both sig_Af + sig_Bf)

After broadcast, both sides monitor for confirmation and the channel is closed.

## VLS Call (Current — 1 round-trip)

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    participant Signer as Bob Signer

    Alice->>Bob: closing_signed(fee, sig_Af)

    Note over Bob: Validates fee + verifies sig_Af (node, not signer)

    Bob->>Signer: SignMutualCloseTx2(<br/>to_local_value_sat=0.2 BTC - fee/2,<br/>to_remote_value_sat=0.8 BTC - fee/2,<br/>local_script=scriptpubkey_B,<br/>remote_script=scriptpubkey_A,<br/>local_wallet_path_hint=[m/...])
    Signer-->>Bob: sig_Bf(close_tx)

    Bob->>Alice: closing_signed(fee, sig_Bf)

    Note over Alice, Bob: Both broadcast close_tx(sig_Af, sig_Bf)
```

### Call Details

| # | Call | Input | Output | Reply used by LDK? |
|---|------|-------|--------|---------------------|
| 1 | SignMutualCloseTx2 | `to_local_value_sat=0.2 BTC - fee/2`, `to_remote_value_sat=0.8 BTC - fee/2`, `local_script=scriptpubkey_B`, `remote_script=scriptpubkey_A`, `local_wallet_path_hint` | `signature` | Yes — goes into `closing_signed` |

---

## Proxy Optimization (v2 — 1 round-trip, no saving)

Same `vls_closing_signed` message as [A01](01-alice-pays-bob-04-close-A01.md). Single call, needed reply, no batching.

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    participant NP as node-proxy
    participant SP as signer-proxy
    participant Signer as Bob Signer

    Alice->>Bob: closing_signed(fee, sig_Af)

    Bob->>NP: SignMutualCloseTx2(to_local=0.2 - fee/2,<br/>to_remote=0.8 - fee/2,<br/>local_script=B, remote_script=A,<br/>path_hint=[m/...])

    NP->>SP: vls_closing_signed(channel_id,<br/>to_local, to_remote,<br/>local_script, remote_script, path_hint)

    SP->>Signer: SignMutualCloseTx2(...)
    Signer-->>SP: sig_Bf(close_tx)

    SP-->>NP: vls_closing_signed_reply(sig_Bf)
    NP-->>Bob: sig_Bf(close_tx)

    Bob->>Alice: closing_signed(fee, sig_Bf)
    Note over Alice, Bob: Both broadcast close_tx(sig_Af, sig_Bf)
```

---

## Channel Closed

After confirmation, the channel is fully settled:
- Alice received 0.8 BTC (minus her fee share) to `scriptpubkey_A`
- Bob received 0.2 BTC (minus his fee share) to `scriptpubkey_B`

Scenario 01 complete.
