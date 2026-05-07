# Lightning Protocol Messages — Routing Node Scenario 1

Lightning wire protocol messages only (peer-to-peer), without VLS signer interactions. Corresponds to the phases in `vls-routing-node-scenario1.md`.

---

## Phase 2: Channel Opening

### Channel 1: Alice funds (inbound, 1,000,000 sat)

```mermaid
sequenceDiagram
    participant Alice
    participant Node
    participant Bitcoin

    Alice->>Node: open_channel(funding=1,000,000 sat)
    Node->>Alice: accept_channel
    Alice->>Node: funding_created(funding_txid, sig)
    Node->>Alice: funding_signed(sig)

    Alice->>Bitcoin: broadcast funding tx
    Note over Alice, Node: wait for confirmation

    Alice->>Node: channel_ready
    Node->>Alice: channel_ready
```

### Channel 2: We fund (outbound, 1,000,000 sat)

```mermaid
sequenceDiagram
    participant Node
    participant Bob
    participant Bitcoin

    Node->>Bob: open_channel(funding=1,000,000 sat)
    Bob->>Node: accept_channel
    Node->>Bob: funding_created(funding_txid, sig)
    Bob->>Node: funding_signed(sig)

    Node->>Bitcoin: broadcast funding tx
    Note over Node, Bob: wait for confirmation

    Node->>Bob: channel_ready
    Bob->>Node: channel_ready
```

---

## Phase 3: Receive and Forward HTLC

Alice pays Bob 100,000 sat through us. Routing fee: 100 sat.

```mermaid
sequenceDiagram
    participant Alice
    participant Node
    participant Bob

    rect rgb(240, 248, 255)
        Note over Alice, Bob: 3.1 — Incoming HTLC on CH1
        Alice->>Node: update_add_htlc(hash=H, amount=100,100 sat, cltv=800,034)
        Alice->>Node: commitment_signed(CH1 #1)
        Node->>Alice: revoke_and_ack
    end

    rect rgb(255, 248, 240)
        Note over Alice, Bob: 3.2 — Forward HTLC on CH2 (fee deducted, CLTV reduced)
        Node->>Bob: update_add_htlc(hash=H, amount=100,000 sat, cltv=800,000)
        Node->>Bob: commitment_signed(CH2 #1)
    end

    rect rgb(255, 248, 240)
        Note over Alice, Bob: 3.3 — Bob revokes old CH2 state
        Bob->>Node: revoke_and_ack
    end

    rect rgb(255, 248, 240)
        Note over Alice, Bob: 3.4 — Bob acknowledges HTLC on CH2
        Bob->>Node: commitment_signed(CH2 #1)
        Node->>Bob: revoke_and_ack
    end

    rect rgb(240, 248, 255)
        Note over Alice, Bob: Sign Alice's updated CH1 commitment
        Node->>Alice: commitment_signed(CH1 #1)
        Alice->>Node: revoke_and_ack
    end
```

---

## Phase 4: HTLC Settlement

Preimage flows back: Bob → Us → Alice.

```mermaid
sequenceDiagram
    participant Alice
    participant Node
    participant Bob

    rect rgb(240, 255, 240)
        Note over Alice, Bob: 4.1 — Bob fulfills HTLC
        Bob->>Node: update_fulfill_htlc(hash=H, preimage=P)
        Bob->>Node: commitment_signed(CH2 #2)
        Node->>Bob: revoke_and_ack
    end

    rect rgb(255, 240, 255)
        Note over Alice, Bob: 4.2 — We fulfill to Alice
        Node->>Alice: update_fulfill_htlc(hash=H, preimage=P)
        Node->>Alice: commitment_signed(CH1 #2)
    end

    rect rgb(255, 240, 255)
        Note over Alice, Bob: 4.3 — Alice acknowledges settlement
        Alice->>Node: revoke_and_ack
        Alice->>Node: commitment_signed(CH1 #2)
        Node->>Alice: revoke_and_ack
    end
```

---

## Phase 6: Cooperative Close

```mermaid
sequenceDiagram
    participant Node
    participant Bob
    participant Bitcoin

    Node->>Bob: shutdown
    Bob->>Node: shutdown
    Node->>Bob: closing_signed(sig)
    Bob->>Node: closing_signed(sig)
    Node->>Bitcoin: broadcast close tx
```

---

## Complete Lifecycle — All Lightning Messages

All peer-to-peer messages for one routed payment, end to end.

```mermaid
sequenceDiagram
    participant Alice
    participant Node
    participant Bob

    rect rgb(240, 248, 255)
        Note over Alice, Bob: Phase 3: Incoming HTLC on CH1

        Alice->>Node: update_add_htlc(H, 100,100 sat, cltv=800,034)
        Alice->>Node: commitment_signed(CH1 #1)
        Node->>Alice: revoke_and_ack
    end

    rect rgb(255, 248, 240)
        Note over Alice, Bob: Phase 3: Outgoing HTLC on CH2

        Node->>Bob: update_add_htlc(H, 100,000 sat, cltv=800,000)
        Node->>Bob: commitment_signed(CH2 #1)
        Bob->>Node: revoke_and_ack
        Bob->>Node: commitment_signed(CH2 #1)
        Node->>Bob: revoke_and_ack

        Node->>Alice: commitment_signed(CH1 #1)
        Alice->>Node: revoke_and_ack
    end

    Note over Alice, Bob: HTLC in flight — waiting for preimage

    rect rgb(240, 255, 240)
        Note over Alice, Bob: Phase 4: Bob fulfills

        Bob->>Node: update_fulfill_htlc(H, preimage)
        Bob->>Node: commitment_signed(CH2 #2)
        Node->>Bob: revoke_and_ack
    end

    rect rgb(255, 240, 255)
        Note over Alice, Bob: Phase 4: We fulfill to Alice

        Node->>Alice: update_fulfill_htlc(H, preimage)
        Node->>Alice: commitment_signed(CH1 #2)
        Alice->>Node: revoke_and_ack
        Alice->>Node: commitment_signed(CH1 #2)
        Node->>Alice: revoke_and_ack
    end

    Note over Alice, Bob: Payment complete — routing fee earned: 100 sat
```
