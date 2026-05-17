# Signer Provisioning via TEE Attestation

This document describes how an operator provisions a remote signer running inside a TEE (Trusted Execution Environment) — delivering the seed and configuration over Nostr, authenticated by a hardware attestation report.

Extends [Nostr Signer Connect (NSC)](nostr-signer-connect.md). See also [Key Derivation](../reference/lightning-key-derivation.md) for what the signer does once it has the seed.

---

## Problem

The signer runs inside an enclave (AWS Nitro, AMD SEV-SNP, or similar). The operator needs to:
1. Verify the signer is running the expected code on genuine hardware
2. Deliver the 24-word seed (and other secrets) to the signer
3. Do this without any party — including the host machine — seeing the seed in transit

The attestation report solves (1) by proving code identity. NIP-44 encryption to a key bound inside the attestation solves (2) and (3).

---

## Architecture

```
  Operator                   Nostr Relay                  Signer (in TEE)
  (npub_owner)               (internal)
     │                           │                            │
     │                           │      ┌──────────────────┐  │
     │                           │      │ 1. Boot enclave  │  │
     │                           │      │ 2. Gen ephemeral │  │
     │                           │      │    nsec/npub     │  │
     │                           │      │ 3. Get attestation│  │
     │                           │      │    (binds npub_e)│  │
     │                           │      └──────────────────┘  │
     │                           │                            │
     │                           │←── kind:23203 ─────────────│  (as npub_ephemeral)
     │                           │    attestation_report      │
     │←── (delivered) ───────────│    {npub_ephemeral bound}  │
     │                           │                            │
     │  Verify:                  │                            │
     │  - HW signature chain    │                            │
     │  - Code measurement      │                            │
     │  - npub_ephemeral bound  │                            │
     │                           │                            │
     │── kind:23204 ────────────→│                            │
     │   NIP-44 encrypted to     │──────────────────────────→│
     │   npub_ephemeral:         │    (delivered)             │
     │   {seed, network}         │                            │
     │                           │      ┌──────────────────┐  │
     │                           │      │ 4. Decrypt seed  │  │
     │                           │      │ 5. Derive:       │  │
     │                           │      │    - npub_signer │  │
     │                           │      │      (permanent) │  │
     │                           │      │    - node_id     │  │
     │                           │      │    - channel keys│  │
     │                           │      │ 6. Discard       │  │
     │                           │      │    ephemeral key │  │
     │                           │      └──────────────────┘  │
     │                           │                            │
     │                           │←── kind:23203 ─────────────│  (as npub_signer)
     │←── (delivered) ───────────│    provisioning_complete   │
     │                           │    {node_id, npub_signer}  │
     │                           │                            │
     │── kind:30078 ────────────→│                            │
     │   Policy binding:         │──────────────────────────→│  (signer subscribes)
     │   tags npub_signer +      │                            │
     │        npub_node           │                            │
     │                           │                            │
     │                           │          NSC operational   │
     │                           │    ┌───────────────────────│
     │                           │    │  Node connects via    │
     │                           │    │  npub_signer from     │
     │                           │    │  kind:30078           │
     │                           │    └───────────────────────│
```

---

## Provisioning Flow

### Step 1: Signer boots inside TEE

The enclave image contains:
- The signer binary (VLS + NSC service)
- The operator's `npub_owner` (hardcoded or in config baked into image)
- The relay URL (hardcoded or in config baked into image)

On first boot, the signer:
1. Generates an **ephemeral** Nostr keypair: `nsec_ephemeral` / `npub_ephemeral` (used only for the provisioning handshake)
2. Connects to the internal relay

### Step 2: Signer requests attestation from hardware

The signer asks the TEE platform for an attestation report, binding `npub_ephemeral` into the report:

**AWS Nitro:**
```
attestation_doc = get_attestation_document(
    public_key = npub_ephemeral [32 bytes],
    user_data  = null,
    nonce      = null
)
// Returns CBOR/COSE_Sign1 signed by Nitro hypervisor
// Contains: PCR0 (enclave image hash), certificate chain → AWS Nitro CA
```

**AMD SEV-SNP:**
```
report_data = SHA256(npub_ephemeral)  // 64 bytes, zero-padded
attestation_report = /dev/sev-guest ioctl SNP_GET_REPORT(report_data)
// Returns binary report signed by VCEK → AMD Root CA
// Contains: MEASUREMENT (launch digest), REPORT_DATA (our hash)
```

The key insight: `npub_ephemeral` is bound *inside* the hardware-signed attestation. No one outside the TEE could have produced this binding. The owner can safely encrypt the seed to this key.

### Step 3: Signer sends attestation to owner

The signer publishes a notification (kind 23203) addressed to `npub_owner`:

```json
{
  "kind": 23203,
  "pubkey": "<npub_ephemeral>",
  "tags": [
    ["p", "<npub_owner>"]
  ],
  "content": "<nip44_encrypted>"
}
```

Encrypted payload:

```json
{
  "notification_type": "attestation_report",
  "data": {
    "platform": "nitro",
    "report": "<base64-encoded attestation document>",
    "npub_ephemeral": "<hex pubkey — also embedded in report>"
  }
}
```

Note: The payload is NIP-44 encrypted to `npub_owner`. The operator must have their `nsec_owner` to decrypt it. The attestation report is also self-authenticating — even if intercepted in plaintext, only the owner knows what measurement to expect.

### Step 4: Owner verifies attestation

The operator (or their provisioning tool) verifies:

1. **Hardware authenticity** — signature chain roots to platform CA (AWS Nitro CA or AMD Root CA)
2. **Code measurement** — PCR0 / MEASUREMENT matches the expected hash of the signer image (the operator built and signed this image, so they know the expected hash)
3. **Key binding** — `npub_ephemeral` in the encrypted payload matches the public key embedded in the attestation report

If all three checks pass, the operator is confident:
- The signer is running the exact code they expect
- On genuine hardware that enforces isolation
- The `npub_ephemeral` key was generated inside that enclave and cannot be extracted
- Anything encrypted to `npub_ephemeral` can only be decrypted inside that enclave

### Step 5: Owner sends seed and configuration

The operator publishes a provisioning event (kind 23204) encrypted to `npub_ephemeral`:

```json
{
  "kind": 23204,
  "pubkey": "<npub_owner>",
  "tags": [
    ["p", "<npub_ephemeral>"],
    ["expiration", "<unix_timestamp>"]
  ],
  "content": "<nip44_encrypted>"
}
```

Encrypted payload:

```json
{
  "method": "provision",
  "params": {
    "seed": "<64-byte hex — BIP39 seed derived from 24 words>",
    "network": "bitcoin"
  }
}
```

| Field | Purpose |
|-------|---------|
| `seed` | The 64-byte BIP39 seed — the signer derives all keys from this (see [key derivation](../reference/lightning-key-derivation.md#from-24-words-to-channel-keys)), **including** the permanent `nsec_signer` |
| `network` | `bitcoin` or `testnet` — determines BIP32 coin type |

### Step 6: Signer derives permanent identity

The signer:
1. Decrypts the provisioning payload using `nsec_ephemeral`
2. Stores the seed in enclave memory (never written to disk, never leaves TEE)
3. Derives the **permanent signer Nostr keypair** from the seed:
   ```
   nsec_signer = BIP32(seed, m/6743'/0')   // deterministic — same seed always yields same npub
   npub_signer = pubkey(nsec_signer)
   ```
4. Derives the Lightning `node_id` from the seed (see key derivation: `m/0'` → `node_secret` → `node_id`)
5. Discards `nsec_ephemeral` — no longer needed
6. Reconnects to the relay as `npub_signer` (the permanent identity)
7. Publishes a provisioning-complete notification to the owner:

```json
{
  "kind": 23203,
  "pubkey": "<npub_signer>",
  "tags": [["p", "<npub_owner>"]],
  "content": "<nip44_encrypted>"
}
```

Encrypted payload:

```json
{
  "notification_type": "provisioning_complete",
  "data": {
    "node_id": "<hex — the Lightning node_id derived from the seed>",
    "npub_signer": "<hex — the permanent signer identity>"
  }
}
```

8. Begins subscribing to kind:23201 (NSC requests) addressed to `npub_signer` — ready for signing operations

**Why derive `nsec_signer` from seed?** Continuity. If the signer reboots or the enclave is restarted, it re-attests with a new ephemeral key, receives the same seed from the owner, and derives the **same** `npub_signer`. All previous relay subscriptions, policy documents, and the node's connection configuration remain valid. No re-pairing needed.

### Step 7: Owner connects signer to node via policy document

The owner publishes a replaceable policy event (kind:30078) that connects `npub_signer` to `npub_node`:

```json
{
  "kind": 30078,
  "pubkey": "<npub_owner>",
  "tags": [
    ["d", "nsc-binding"],
    ["p", "<npub_signer>", "signer"],
    ["p", "<npub_node>", "node"]
  ],
  "content": "<nip44_encrypted or plaintext>"
}
```

Content (policy configuration):

```json
{
  "relay": "ws://relay.internal:7777",
  "allowlist": ["<peer_pubkey_1>", "<peer_pubkey_2>"],
  "max_channel_value_sat": 100000000,
  "usage_profile_npub": "<npub_owner>"
}
```

Both the signer and the node subscribe to kind:30078 events from `npub_owner`. When each sees this binding event:
- **Signer** learns: "I should accept NSC requests from `npub_node`"
- **Node** learns: "My signer is at `npub_signer` on this relay"

This replaces the manual connection URI delivery. The owner's policy document is the single source of truth that binds signer to node.

### Step 8: Node connects to signer

The node, having observed the kind:30078 binding event:
1. Derives its own `nsec_node` (from its own seed or configuration)
2. Computes the NIP-44 conversation key with `npub_signer`
3. Subscribes to kind:23202 (responses) addressed to `npub_node`
4. Begins sending NSC requests

The system is now operational.

---

## Event Kinds for Provisioning

| Kind | Direction | Purpose |
|------|-----------|---------|
| 23203 | Signer (ephemeral) → Owner | Attestation report |
| 23204 | Owner → Signer (ephemeral) | Seed delivery |
| 23203 | Signer (permanent) → Owner | Provisioning complete |
| 30078 | Owner → all | Policy / NSC binding (replaceable) |

- Kind 23203 (NSC notification) is reused for signer→owner messages.
- Kind 23204 is provisioning-specific — carries the seed encrypted to the attested ephemeral key.
- Kind 30078 is the standard replaceable policy event that binds signer to node. Both sides subscribe to it.

---

## Security Properties

### What the host/relay cannot learn

| Secret | Protected by |
|--------|-------------|
| Seed (24 words) | NIP-44 encryption to `npub_ephemeral` (key inside TEE) |
| Derived private keys | Never leave TEE memory |
| `nsec_signer` | Derived from seed inside TEE, never exported |
| `nsec_ephemeral` | Generated inside TEE, discarded after provisioning |
| Provisioning payload | Encrypted end-to-end owner↔signer |

### What an attacker would need to compromise

| Attack | Requires |
|--------|----------|
| Forge attestation | Break platform CA signature (infeasible) |
| Extract seed from TEE | Break hardware isolation (enclave escape — rare, patchable) |
| MITM provisioning | Compromise `nsec_owner` (operator's key) |
| Replay provisioning | Expired — `expiration` tag on the event |
| Swap enclave image | Different measurement — attestation fails verification |

### Defense in depth

Even if the relay is compromised:
- Attestation report is signed by hardware, not the relay
- Seed is encrypted to a key inside the TEE
- The relay cannot forge events from `npub_owner` (doesn't have `nsec_owner`)

---

## Re-provisioning and Rotation

### Signer restart (same image)

On restart, the signer:
1. Generates a **new** `nsec_ephemeral` (different from last boot)
2. Re-attests with the new ephemeral key bound in the report
3. Sends attestation to owner
4. Owner verifies (same measurement — same image) and re-sends the seed
5. Signer derives the **same** `npub_signer` from the seed (deterministic)
6. Resumes operations — the node doesn't notice the restart (same `npub_signer`, same relay)

The seed is never persisted to disk — it must be re-delivered after every cold restart. This is a feature: if the enclave is compromised, simply stopping it destroys the seed. But from the node's perspective, nothing changes — same signer identity, no re-pairing required.

### Image upgrade

1. Operator builds new signer image → new measurement hash
2. Operator updates their expected-measurement config
3. New enclave boots → attests with new measurement → operator verifies → re-provisions same seed
4. Signer derives same `npub_signer` → resumes on the same identity

The seed doesn't change — only the enclave image changes. Same 24 words, same keys, same channels, same `npub_signer`.

### Key rotation (new seed)

A new seed means a new `npub_signer` (since it's derived from the seed). This requires:
1. Closing all existing channels
2. Provisioning the new seed → new `npub_signer`
3. Owner publishes updated kind:30078 binding event (new `npub_signer`, same `npub_node`)
4. Re-opening channels

This is a heavyweight operation — only done if the seed is suspected compromised.

---

## Comparison: Manual vs Attested Provisioning

| | Manual | Attested (this doc) |
|---|---|---|
| Seed delivery | Operator types seed into signer console | Encrypted Nostr event to attested key |
| Code verification | Operator trusts the deployment | Hardware-signed measurement |
| Signer identity | Configured manually, changes on reboot | Deterministic from seed — survives restarts |
| Node↔signer binding | Operator copies connection URI | Owner publishes kind:30078 policy event |
| Remote operation | Requires console/SSH access | Only requires Nostr relay connectivity |
| Auditability | No proof of what code received the seed | Attestation report is cryptographic evidence |
| Restart recovery | Manual re-configuration | Automatic — re-attest, re-provision, same npub |

---

## Open Questions

1. **Seed vs mnemonic** — should the operator send the raw 64-byte seed or the 24 mnemonic words? The seed is more general (no BIP39 dependency), but the mnemonic is what operators actually back up.

2. **Warm restart (sealed storage)** — should the signer be able to persist the seed encrypted to the TEE's sealing key so it can restart without re-provisioning? This trades "stop = destroy" security for operational convenience. AWS Nitro doesn't support sealing; AMD SEV-SNP does via VMSA-based sealing.

3. **Multi-owner quorum** — could provisioning require M-of-N owners to each attest their approval before the seed is delivered? This would use Nostr-native multi-sig or threshold schemes.

4. **Attestation refresh** — should the signer periodically re-attest and publish fresh reports? This would detect runtime compromise (e.g., if the TEE is hot-patched).

5. **Signer Nostr derivation path** — the document uses `m/6743'/0'` as a placeholder. Should this be standardized? It must not collide with existing paths (`m/0'` node key, `m/3'` channel master).

6. **Policy document encryption** — should the kind:30078 binding event content be NIP-44 encrypted (only signer and node can read it) or plaintext (simpler, but exposes the allowlist and channel limits)?

7. **Automatic re-provisioning** — should the owner's provisioning tool automatically re-send the seed when it sees a new attestation from a known-good measurement? This enables unattended restarts at the cost of reducing the owner's control over when the seed is active.
