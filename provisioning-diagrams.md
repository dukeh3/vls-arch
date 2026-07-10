# Provisioning Flow Diagrams

## Mission 6: Initiator-Based Key Provisioning

The implemented provisioning flow from [mission-6-architecture](../workspaces/vls-signer-wks/plans/mission-6-architecture.md). The initiator holds the seed and signer identity, verifies the SNP attestation, and provisions the signer over Nostr (kind 23210).

```mermaid
sequenceDiagram
    participant I as Initiator<br/>(Docker, bi_pub)
    participant R as Nostr Relay
    participant S as Signer<br/>(SNP Enclave)

    Note over S: VM boots

    S->>S: Generate key_start<br/>(ephemeral Nostr keypair)
    S->>S: Get SNP attestation report<br/>(key_start pubkey in report_data)

    S->>R: Attestation event (kind 1)<br/>signed by key_start, tagged bi_pub<br/>{report, product}
    R->>I: Deliver attestation event

    Note over I: Verify attestation
    I->>I: Check SNP report signature<br/>(VCEK -> ASK -> ARK)
    I->>I: Check report_data contains<br/>key_start pubkey
    I->>I: Check policy: debug=0,<br/>correct VMPL

    I->>R: Provisioning event (kind 23210)<br/>NIP-04 encrypted to key_start<br/>{seed, proxy_pubkey,<br/>signer_pubkey, signer_nsec}
    R->>S: Deliver provisioning event

    S->>S: Decrypt with key_start
    S->>S: Initialize VLS with seed
    S->>S: Switch identity to bs_pub

    Note over S: Start signing loop<br/>(accept requests from bn_pub)

    S->>R: Subscribe kind 23201<br/>from bn_pub
```

## TEE Attestation Provisioning (Future Design)

The full provisioning flow from [signer-provisioning](future/signer-provisioning.md). The signer derives its permanent identity from the seed (deterministic), and the owner publishes a policy binding event (kind 30078) to connect signer to node.

```mermaid
sequenceDiagram
    participant O as Owner<br/>(npub_owner)
    participant R as Nostr Relay
    participant S as Signer<br/>(TEE Enclave)
    participant N as Node

    Note over S: Enclave boots

    S->>S: Generate nsec_ephemeral<br/>(temporary keypair)
    S->>S: Request TEE attestation<br/>(npub_ephemeral bound<br/>in report)

    S->>R: kind 23203 (as npub_ephemeral)<br/>NIP-44 encrypted to npub_owner<br/>{attestation_report,<br/>npub_ephemeral}
    R->>O: Deliver attestation

    Note over O: Verify attestation
    O->>O: HW signature chain<br/>(platform CA)
    O->>O: Code measurement<br/>(expected image hash)
    O->>O: npub_ephemeral matches<br/>key in report

    O->>R: kind 23204<br/>NIP-44 encrypted to npub_ephemeral<br/>{seed, network}
    R->>S: Deliver seed

    S->>S: Decrypt with nsec_ephemeral
    S->>S: Derive from seed:<br/>nsec_signer (m/6743'/0')<br/>node_id (m/0')
    S->>S: Discard nsec_ephemeral

    S->>R: kind 23203 (as npub_signer)<br/>NIP-44 encrypted to npub_owner<br/>{provisioning_complete,<br/>node_id, npub_signer}
    R->>O: Confirm provisioning

    O->>R: kind 30078 (replaceable)<br/>Policy binding:<br/>npub_signer + npub_node
    R->>S: Deliver policy binding
    R->>N: Deliver policy binding

    Note over S: Subscribe kind 23201<br/>(signing requests)
    Note over N: Learns signer identity<br/>from kind 30078

    N->>R: kind 23201 (signing request)<br/>NIP-44 encrypted to npub_signer
    R->>S: Deliver request
    S->>R: kind 23202 (signing response)
    R->>N: Deliver response

    Note over N,S: Operational
```

## Key Differences

| Aspect | Mission 6 | Future Design |
|--------|-----------|---------------|
| Signer identity | Provisioned by initiator (bs_pub/bs_prv sent explicitly) | Derived from seed (deterministic, m/6743'/0') |
| Attestation kind | kind 1 (text note) | kind 23203 (dedicated) |
| Seed delivery kind | kind 23210 | kind 23204 |
| Node-signer binding | Hardcoded proxy pubkey | Owner publishes kind 30078 policy event |
| Provisioning payload | seed + proxy_pubkey + signer_pubkey + signer_nsec | seed + network |
| Restart behavior | Re-attest, re-provision, same identity (explicitly sent) | Re-attest, re-provision, same identity (derived from seed) |
| Encryption | NIP-04 | NIP-44 |
