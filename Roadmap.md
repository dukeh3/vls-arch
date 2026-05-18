# Roadmap

From documentation baseline to production signer in a secure enclave.

---

## Milestone 1: Architecture 0.0.1 (Documentation Baseline)

**What exists today:** VLS as a single signer with separate API calls — one call per operation, no batching, no proxy layer. The node talks directly to the signer over vsock or TCP.

**What this milestone produced:**

- **reference/** — protocol-level material independent of VLS: Lightning message flow, key derivation notation, scenario roadmap
- **current/** — VLS signer calls as they work today, with every call documented against the source (commit `75e3a46b`, protocol version 6)
- **Scenario coverage:**
  - [01 — Alice Pays Bob](current/01-alice-pays-bob.md) — open, HTLC add, HTLC settle, cooperative close
  - [02 — Alice Pays Bob's BOLT11 Invoice](current/02-alice-pays-bobs-bolt11-invoice.md) — invoice-driven payment with `PreapproveInvoice`
- **Policy rules** — [vls-policy-rules.md](current/vls-policy-rules.md) documents the full set of signer policy checks (on-chain, commitment, mutual close, sweep)

**The seam is identified:** the point between node and signer where the proxy will sit. The `current/` docs establish every call that crosses that boundary, making the compression targets visible.

---

## Milestone 2: 2-Proxy Setup (v2 Named Protocol)

**What changes:** A node-proxy and signer-proxy sit on either side of the trust boundary. They speak a semantic protocol of named messages — one round-trip per Lightning wire message — while the node and signer APIs remain unchanged.

```
  Node ──→ node-proxy ════→ signer-proxy ──→ Signer
           (buffers,   named    (unpacks,
            translates) msgs     dispatches)
```

**Call compression on the cross-trust-boundary link:**

| Phase | Current (per side) | v2 (per side) |
|-------|-------------------|---------------|
| Channel open | 7-10 calls | **5 RTTs** |
| Each commitment update | 3 calls | **3 RTTs** |
| Cooperative close | 1 call | **1 RTT** |
| **Full scenario 01** | **~19 calls** | **12 RTTs** |

The main win is channel open — independent calls (NewChannel + GetChannelBasepoints + GetPerCommitmentPoint) fold into a single `vls_create_channel` message. Commitment updates are already minimal; the proxy adds deferral (combining ValidateCommitmentTx2 + RevokeCommitmentTx into one `vls_revoke_commitment` RTT) without reducing the per-state-change count.

**Messages are bounded by protocol moments** — each proxy message fires at exactly the point where a Lightning wire message must be sent or received. No artificial grouping.

**Documented in:** [future/01-alice-pays-bob.md](future/01-alice-pays-bob.md) and per-step files (01-open-channel, 02-htlc-add, 04-close).

---

## Milestone 3: Secure Enclave in AWS (TEE + NSC)

**What changes:** The signer runs inside an AWS Nitro Enclave. The proxy-to-proxy link runs over [Nostr Signer Connect (NSC)](future/nostr-signer-connect.md) — VLS protocol messages carried as NIP-44-encrypted Nostr events on an internal relay.

```
  Node ──→ node-proxy ──→ [Internal Relay] ──→ signer-proxy ──→ Signer (Nitro Enclave)
                            NIP-AB access        NSC events
                            control              (kind 23201/23202)
```

### TEE attestation-based provisioning

The signer never receives the seed in plaintext outside the enclave. The [provisioning flow](future/signer-provisioning.md):

1. Signer boots in enclave, generates ephemeral Nostr keypair
2. Requests hardware attestation binding `npub_ephemeral` into the report (PCR0 + certificate chain to AWS Nitro CA)
3. Sends attestation to owner via relay (kind 23203)
4. Owner verifies hardware authenticity, code measurement, and key binding
5. Owner encrypts seed to `npub_ephemeral` via NIP-44 (kind 23204) — only the enclave can decrypt
6. Signer derives permanent `npub_signer` from seed (deterministic — survives restarts)
7. Owner publishes kind:30078 policy event binding `npub_signer` to `npub_node`

### NSC as transport

NSC is one of three Nostr-based Lightning protocols sharing the same pattern:

| Protocol | Connects | Purpose |
|----------|----------|---------|
| NWC (NIP-47) | App → Node | Wallet operations |
| NNC (NIP-XX) | Owner → Node | Node administration |
| NSC | Node → Signer | VLS signing operations |

All three run on the same internal relay with separate event kinds. The signer processes compressed v2 proxy messages — the same named protocol from Milestone 2, now transported as Nostr events instead of raw TCP/vsock.

**Documented in:** [future/nostr-signer-connect.md](future/nostr-signer-connect.md), [future/signer-provisioning.md](future/signer-provisioning.md)

---

## Milestone 4: AMD SEV-SNP Platform

**What changes:** The same signer, proxy protocol, and NSC transport from Milestone 3 — but running on AMD SEV-SNP instead of AWS Nitro. This removes the AWS dependency and enables bare-metal or multi-cloud deployments.

### What differs from Nitro

| Aspect | AWS Nitro (Milestone 3) | AMD SEV-SNP |
|--------|------------------------|-------------|
| Attestation report | CBOR/COSE_Sign1 signed by Nitro hypervisor | Binary report signed by VCEK → AMD Root CA |
| Code measurement | PCR0 (enclave image hash) | MEASUREMENT (launch digest) |
| Key binding | `public_key` field in attestation doc | `REPORT_DATA = SHA256(npub_ephemeral)` |
| Attestation API | `get_attestation_document()` | `/dev/sev-guest` ioctl `SNP_GET_REPORT` |
| Sealed storage | Not supported — seed must be re-delivered on every restart | Supported via VMSA-based sealing — optional warm restart without re-provisioning |
| Hosting | AWS only | Any AMD EPYC host — bare metal, Hetzner, Azure, GCP |

### Provisioning flow

The [provisioning protocol](future/signer-provisioning.md) is identical — only the attestation verification step changes:

1. Owner checks signature chain roots to **AMD Root CA** (not AWS Nitro CA)
2. Owner checks **MEASUREMENT** (not PCR0) against expected launch digest
3. Owner checks `SHA256(npub_ephemeral)` matches **REPORT_DATA** in the report

Everything else — ephemeral key generation, NIP-44 seed delivery, permanent `npub_signer` derivation, kind:30078 policy binding — works unchanged.

### Sealed storage option

AMD SEV-SNP can seal the seed to the platform so the signer survives restarts without re-provisioning. This trades the "stop = destroy seed" property of Nitro for operational convenience. The choice is per-deployment — the provisioning protocol supports both modes.

### Why this matters

AWS Nitro ties the signer to a single cloud provider. AMD SEV-SNP runs on commodity hardware, enabling self-hosted deployments where the operator controls the physical machine. For a signer guarding Lightning channel funds, minimizing third-party dependencies is worth the additional platform work.

---

## Beyond: v3-v5 and Multi-Signer

Once the relay is in place, the signer can subscribe to more than just NSC requests:

| Version | What the proxy pair adds |
|---------|--------------------------|
| **v1** | Single inline passthrough proxy (the structural seam) |
| **v2** | Named proxy messages with call compression (Milestone 2) |
| **v3** | Signer-proxy ingests NWC/NNC events for `kind:30078` UsageProfile enforcement |
| **v4** | Signer-proxy ingests TXOO chain attestations alongside proxy messages |
| **v5** | Proxy-to-proxy link runs over the internal Nostr relay (Milestone 3 / NSC) |

Each step extends one of the two proxies; the node, the signer, and the BOLT-level VLS protocol stay stable.

### Multi-signer and Taproot (Epics 2-3)

| Epic | Lightning | Signer | Status |
|------|-----------|--------|--------|
| 2 | BOLT 3 (script-based) | Multi-signer (quorum/threshold) | Not started |
| 3 | Taproot (MuSig2/tapscript) | Multi-signer | Not started |

These are orthogonal to the proxy/transport roadmap — they change what the signer does, not how it connects.
