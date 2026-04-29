# VLS Protocol Messages — Complete Reference

This document specifies all request/response messages exchanged between a Lightning node and the VLS (Validating Lightning Signer). The protocol uses BOLT-style binary serialization with length-prefixed framing (u32 big-endian length + u16 message type + payload).

## Protocol Versioning

| Constant | Value | Description |
|----------|-------|-------------|
| MIN_PROTOCOL_VERSION | 2 | Minimum supported version |
| DEFAULT_MAX_PROTOCOL_VERSION | 6 | Current default max (PROTOCOL_VERSION_NO_SECRET) |
| PROTOCOL_VERSION_REVOKE | 5 | Introduced `RevokeCommitmentTx` as a separate message |
| PROTOCOL_VERSION_NO_SECRET | 6 | `GetPerCommitmentPoint` no longer returns old secret |

Source: `vls-protocol/src/msgs.rs`

---

## Common Data Types

| Type | Size | Description |
|------|------|-------------|
| `Secret` | 32 bytes | Sensitive secret (node key material) |
| `DisclosedSecret` | 32 bytes | Revocation secret (safe to share after revocation) |
| `DevSecret` | 32 bytes | Development/test secret |
| `PubKey` | 33 bytes | Compressed secp256k1 public key |
| `Signature` | 64 bytes | ECDSA signature (r, s) |
| `RecoverableSignature` | 65 bytes | ECDSA signature with recovery ID |
| `BitcoinSignature` | 65 bytes | Signature (64) + sighash type (1) |
| `Sha256` | 32 bytes | SHA-256 hash |
| `ExtKey` | 78 bytes | BIP32 extended key |
| `Txid` | 32 bytes | Transaction ID |
| `BlockHash` | 32 bytes | Block header hash |
| `Basepoints` | 4 x PubKey | revocation, payment, htlc, delayed_payment |
| `Htlc` | variable | side (u8), amount (u64), payment_hash (32), cltv_expiry (u32) |
| `Utxo` | variable | txid, outnum, amount, keyindex, is_p2sh, script, close_info |
| `WireString` | variable | Length-prefixed UTF-8 string |
| `Octets` | variable | Length-prefixed raw bytes |
| `WithSize<T>` | variable | Size-prefixed serialized object |

---

## Table of Contents

1. [Initialization](#1-initialization)
2. [Channel Lifecycle](#2-channel-lifecycle)
3. [Commitment Transactions — Holder](#3-commitment-transactions--holder)
4. [Commitment Transactions — Counterparty](#4-commitment-transactions--counterparty)
5. [Revocation](#5-revocation)
6. [Per-Commitment Points](#6-per-commitment-points)
7. [Mutual Close](#7-mutual-close)
8. [HTLC & Sweep Signing](#8-htlc--sweep-signing)
9. [CLN Any-Channel Dispatch](#9-cln-any-channel-dispatch)
10. [Gossip & Announcements](#10-gossip--announcements)
11. [Invoice Signing](#11-invoice-signing)
12. [BOLT12 Signing](#12-bolt12-signing)
13. [On-chain / PSBT Signing](#13-on-chain--psbt-signing)
14. [Key Derivation & Verification](#14-key-derivation--verification)
15. [Elliptic Curve Operations](#15-elliptic-curve-operations)
16. [Blockchain Synchronization](#16-blockchain-synchronization)
17. [Miscellaneous](#17-miscellaneous)
18. [Error Handling](#18-error-handling)
19. [Protocol Flows](#19-protocol-flows)

---

## 1. Initialization

These messages establish the signer session, negotiate protocol version, and return the node's identity.

---

### HsmdDevPreinit — Developer Pre-Initialization

| | |
|---|---|
| **ID** | 90 |
| **Direction** | Node → Signer |
| **Compatibility** | Developer mode only |
| **When sent** | Before `HsmdInit`, to configure dev/test settings |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `derivation_style` | u8 | Key derivation style (0=native, 1=lnd) |
| `network_name` | WireString | Bitcoin network ("regtest", "testnet", "bitcoin") |
| `seed` | Option\<DevSecret\> | Custom seed for deterministic testing |
| `allowlist` | Array\<WireString\> | Allowlisted destination addresses |

**Response: HsmdDevPreinitReply (ID: 190)**

| Field | Type | Description |
|-------|------|-------------|
| `node_id` | PubKey | The signer's node public key |

---

### HsmdDevPreinit2 — Developer Pre-Initialization V2

| | |
|---|---|
| **ID** | 99 |
| **Direction** | Node → Signer |
| **Compatibility** | Developer mode only |
| **When sent** | Alternative to HsmdDevPreinit with TLV-encoded options |

**Request fields (TLV-encoded):**

| Tag | Field | Type | Description |
|-----|-------|------|-------------|
| 1 | `fail_preapprove` | Option\<bool\> | Force preapproval failures |
| 3 | `no_preapprove_check` | Option\<bool\> | Skip preapproval checks |
| 252 | `derivation_style` | Option\<u8\> | Key derivation style |
| 251 | `network_name` | Option\<WireString\> | Network name |
| 250 | `seed` | Option\<DevSecret\> | Custom seed |
| 249 | `allowlist` | Option\<Array\<WireString\>\> | Allowlisted addresses |

**Response:** None (no reply message)

---

### HsmdInit — Initialize Signer (CLN)

| | |
|---|---|
| **ID** | 11 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | At node startup, first message after optional dev preinit |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `key_version` | Bip32KeyVersion | BIP32 public/private key version bytes |
| `chain_params` | BlockHash | Network genesis block hash |
| `encryption_key` | Option\<DevSecret\> | HSM encryption key (dev) |
| `dev_privkey` | Option\<DevPrivKey\> | Override node private key (dev) |
| `dev_bip32_seed` | Option\<DevSecret\> | Override BIP32 seed (dev) |
| `dev_channel_secrets` | Option\<Array\<DevSecret\>\> | Override channel secrets (dev) |
| `dev_channel_secrets_shaseed` | Option\<Sha256\> | Override SHA seed (dev) |
| `hsm_wire_min_version` | u32 | Minimum protocol version node accepts |
| `hsm_wire_max_version` | u32 | Maximum protocol version node accepts |

**Response: HsmdInitReplyV2 (ID: 113)** — when negotiated protocol_version < 4

| Field | Type | Description |
|-------|------|-------------|
| `node_id` | PubKey | Node public key |
| `bip32` | ExtKey | Account-level extended public key |
| `bolt12` | PubKey | BOLT12 signing key |

**Response: HsmdInitReplyV4 (ID: 114)** — when negotiated protocol_version >= 4

| Field | Type | Description |
|-------|------|-------------|
| `hsm_version` | u32 | Negotiated protocol version |
| `hsm_capabilities` | ArrayBE\<u32\> | Supported message type IDs |
| `node_id` | PubKey | Node public key |
| `bip32` | ExtKey | Account-level extended public key |
| `bolt12` | PubKey | BOLT12 signing key |

---

### HsmdInit2 — Initialize Signer (LDK)

| | |
|---|---|
| **ID** | 1011 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | At node startup |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `derivation_style` | u8 | Key derivation style |
| `network_name` | WireString | Bitcoin network name |
| `dev_seed` | Option\<DevSecret\> | Custom seed (dev) |
| `dev_allowlist` | Array\<WireString\> | Allowlisted addresses (dev) |

**Response: HsmdInit2Reply (ID: 1111)**

| Field | Type | Description |
|-------|------|-------------|
| `node_id` | PubKey | Node public key |
| `bip32` | ExtKey | Account-level extended public key |
| `bolt12` | PubKey | BOLT12 signing key |

---

### NodeInfo — Get Node Information (LDK)

| | |
|---|---|
| **ID** | 1012 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Any time to query node identity |

**Request fields:** None

**Response: NodeInfoReply (ID: 1112)**

| Field | Type | Description |
|-------|------|-------------|
| `network_name` | WireString | Bitcoin network |
| `node_id` | PubKey | Node public key |
| `bip32` | ExtKey | Account-level extended public key |

---

## 2. Channel Lifecycle

These messages manage the creation, configuration, and teardown of channels in the signer's state.

---

### NewChannel — Register New Channel

| | |
|---|---|
| **ID** | 30 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | When a new channel is being opened or restored from database |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Database ID for this channel |

**Response: NewChannelReply (ID: 130)** — Empty

---

### SetupChannel — Configure Channel Parameters

| | |
|---|---|
| **ID** | 31 |
| **Direction** | Node → Signer |
| **Compatibility** | Both CLN and LDK |
| **When sent** | After NewChannel (CLN) or when establishing a channel (LDK), providing full channel configuration |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `is_outbound` | bool | We initiated (funded) the channel |
| `channel_value` | u64 | Total channel capacity in satoshis |
| `push_value` | u64 | Initial push to counterparty in millisatoshis |
| `funding_txid` | Txid | Funding transaction ID |
| `funding_txout` | u16 | Funding output index |
| `to_self_delay` | u16 | Our CSV delay (blocks) |
| `local_shutdown_script` | Octets | Our upfront shutdown script |
| `local_shutdown_wallet_index` | Option\<u32\> | Wallet derivation index for shutdown script |
| `remote_basepoints` | Basepoints | Counterparty's 4 channel basepoints |
| `remote_funding_pubkey` | PubKey | Counterparty's funding public key |
| `remote_to_self_delay` | u16 | Counterparty's CSV delay (blocks) |
| `remote_shutdown_script` | Octets | Counterparty's upfront shutdown script |
| `channel_type` | Octets | Negotiated channel type feature bits |

**Response: SetupChannelReply (ID: 131)** — Empty

---

### CheckOutpoint — Verify Funding Confirmation

| | |
|---|---|
| **ID** | 32 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | After funding transaction is broadcast, to check confirmation depth |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `funding_txid` | Txid | Funding transaction ID |
| `funding_txout` | u16 | Funding output index |

**Response: CheckOutpointReply (ID: 132)**

| Field | Type | Description |
|-------|------|-------------|
| `is_buried` | bool | Funding has reached sufficient confirmation depth |

---

### LockOutpoint — Lock Funding UTXO

| | |
|---|---|
| **ID** | 37 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | After funding is confirmed, to prevent accidental double-spend |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `funding_txid` | Txid | Funding transaction ID |
| `funding_txout` | u16 | Funding output index |

**Response: LockOutpointReply (ID: 137)** — Empty

---

### ForgetChannel — Remove Channel State

| | |
|---|---|
| **ID** | 34 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | After channel is fully closed and resolved |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `node_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: ForgetChannelReply (ID: 134)** — Empty

---

## 3. Commitment Transactions — Holder

These messages validate and sign the holder's (our) commitment transaction.

---

### ValidateCommitmentTx — Validate Holder Commitment (CLN)

| | |
|---|---|
| **ID** | 35 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | When receiving `commitment_signed` from the counterparty — the node passes the counterparty's signature for our commitment to the signer for validation |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | Our commitment transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `htlcs` | Array\<Htlc\> | All HTLCs in the commitment |
| `commitment_number` | u64 | Commitment sequence number |
| `feerate` | u32 | Feerate in sat/kw |
| `signature` | BitcoinSignature | Counterparty's signature on our commitment |
| `htlc_signatures` | Array\<BitcoinSignature\> | Counterparty's signatures on HTLC transactions |

**Response: ValidateCommitmentTxReply (ID: 135)**

| Field | Type | Description |
|-------|------|-------------|
| `old_commitment_secret` | Option\<DisclosedSecret\> | Revocation secret for previous commitment (only if protocol_version < 5) |
| `next_per_commitment_point` | PubKey | Per-commitment point for the next commitment |

**What the signer does:**
1. Reconstructs the expected commitment from its own state
2. Validates all policy rules (structure, values, fees, keys, HTLCs)
3. Verifies the counterparty's signature is valid
4. Advances the commitment state machine
5. Returns the next per-commitment point (and old secret if protocol < 5)

---

### ValidateCommitmentTx2 — Validate Holder Commitment (LDK)

| | |
|---|---|
| **ID** | 1035 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Same as ValidateCommitmentTx, but with value-based fields instead of full transaction |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment sequence number |
| `feerate` | u32 | Feerate in sat/kw |
| `to_local_value_sat` | u64 | Our balance in satoshis |
| `to_remote_value_sat` | u64 | Counterparty balance in satoshis |
| `htlcs` | Array\<Htlc\> | All HTLCs |
| `signature` | BitcoinSignature | Counterparty's signature |
| `htlc_signatures` | Array\<BitcoinSignature\> | Counterparty's HTLC signatures |

**Response: ValidateCommitmentTxReply (ID: 135)** — Same as above

---

### SignLocalCommitmentTx2 — Sign Holder Commitment (LDK)

| | |
|---|---|
| **ID** | 1005 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Phase 2 signing of our own commitment (after validation) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number to sign |

**Response: SignCommitmentTxReply (ID: 105)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature on our commitment |

---

### SignCommitmentTx — Sign Commitment or Mutual Close (CLN)

| | |
|---|---|
| **ID** | 5 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | To sign our commitment transaction (CLN uses this for both commitment and some close scenarios) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |
| `tx` | WithSize\<Transaction\> | Transaction to sign |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `remote_funding_key` | PubKey | Counterparty's funding key |
| `commitment_number` | u64 | Commitment number |

**Response: SignCommitmentTxReply (ID: 105)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature |

---

## 4. Commitment Transactions — Counterparty

These messages sign the counterparty's commitment transaction (we provide our signature to the peer).

---

### SignRemoteCommitmentTx — Sign Counterparty Commitment (CLN)

| | |
|---|---|
| **ID** | 19 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | When we want to send `commitment_signed` to the peer — we need our signature on their commitment |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | Counterparty's commitment transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `remote_funding_key` | PubKey | Counterparty's funding key |
| `remote_per_commitment_point` | PubKey | Counterparty's per-commitment point |
| `option_static_remotekey` | bool | Static remote key feature enabled |
| `commitment_number` | u64 | Commitment number |
| `htlcs` | Array\<Htlc\> | All HTLCs in the commitment |
| `feerate` | u32 | Feerate in sat/kw |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature on counterparty's commitment |

---

### SignRemoteCommitmentTx2 — Sign Counterparty Commitment (LDK)

| | |
|---|---|
| **ID** | 1019 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Same purpose as SignRemoteCommitmentTx, returns both commitment and HTLC signatures |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `remote_per_commitment_point` | PubKey | Counterparty's per-commitment point |
| `commitment_number` | u64 | Commitment number |
| `feerate` | u32 | Feerate in sat/kw |
| `to_local_value_sat` | u64 | Counterparty's balance |
| `to_remote_value_sat` | u64 | Our balance |
| `htlcs` | Array\<Htlc\> | All HTLCs |

**Response: SignCommitmentTxWithHtlcsReply (ID: 1119)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature on their commitment |
| `htlc_signatures` | Array\<BitcoinSignature\> | Our signatures on their HTLC transactions |

---

## 5. Revocation

These messages handle commitment revocation — the mechanism that makes old states punishable.

---

### RevokeCommitmentTx — Revoke Previous Commitment

| | |
|---|---|
| **ID** | 40 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only (protocol_version >= 5) |
| **When sent** | After receiving and validating a new commitment, to revoke the old one and obtain its secret for sending `revoke_and_ack` to the peer |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | The commitment number being revoked |

**Response: RevokeCommitmentTxReply (ID: 140)**

| Field | Type | Description |
|-------|------|-------------|
| `old_commitment_secret` | DisclosedSecret | Revocation secret for the revoked commitment |
| `next_per_commitment_point` | PubKey | Next per-commitment point |

**Note:** Before protocol_version 5, revocation was implicit in `ValidateCommitmentTx` response.

---

### ValidateRevocation — Accept Counterparty's Revocation

| | |
|---|---|
| **ID** | 36 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | When receiving `revoke_and_ack` from the peer — pass their revocation secret to the signer for validation and storage |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | The commitment number being revoked |
| `commitment_secret` | DisclosedSecret | Counterparty's revocation secret |

**Response: ValidateRevocationReply (ID: 136)** — Empty

**What the signer does:**
1. Verifies the secret matches the expected value for this commitment number
2. Stores the secret (enables penalty transaction if counterparty broadcasts revoked state)
3. Advances the counterparty revocation state

---

### CheckFutureSecret — Pre-validate Commitment Secret

| | |
|---|---|
| **ID** | 22 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | To verify a future commitment secret is correct before it's needed |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Future commitment number |
| `secret` | DisclosedSecret | Secret to validate |

**Response: CheckFutureSecretReply (ID: 122)**

| Field | Type | Description |
|-------|------|-------------|
| `result` | bool | Secret is valid for this commitment number |

---

## 6. Per-Commitment Points

---

### GetPerCommitmentPoint — Get Commitment Point (CLN)

| | |
|---|---|
| **ID** | 18 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | To retrieve the per-commitment point for a specific commitment number |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number |

**Response: GetPerCommitmentPointReply (ID: 118)**

| Field | Type | Description |
|-------|------|-------------|
| `point` | PubKey | Per-commitment point |
| `secret` | Option\<DisclosedSecret\> | Old secret (if commitment_number >= 2 and protocol_version < 6) |

---

### GetPerCommitmentPoint2 — Get Commitment Point (LDK)

| | |
|---|---|
| **ID** | 1018 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Same as GetPerCommitmentPoint, without returning old secret |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number |

**Response: GetPerCommitmentPoint2Reply (ID: 1118)**

| Field | Type | Description |
|-------|------|-------------|
| `point` | PubKey | Per-commitment point |

---

## 7. Mutual Close

---

### SignMutualCloseTx — Sign Cooperative Close (CLN)

| | |
|---|---|
| **ID** | 21 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | During cooperative close negotiation |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | Mutual close transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `remote_funding_key` | PubKey | Counterparty's funding key |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature on the close transaction |

**What the signer validates:**
- Destination is allowlisted or in our wallet
- Output value matches our expected balance
- Fee is within acceptable range
- No pending HTLCs remain

---

### SignMutualCloseTx2 — Sign Cooperative Close (LDK)

| | |
|---|---|
| **ID** | 1021 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | During cooperative close |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `to_local_value_sat` | u64 | Our output value |
| `to_remote_value_sat` | u64 | Counterparty's output value |
| `local_script` | Octets | Our shutdown script |
| `remote_script` | Octets | Counterparty's shutdown script |
| `local_wallet_path_hint` | ArrayBE\<u32\> | Wallet derivation path hint |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature |

---

## 8. HTLC & Sweep Signing

These messages sign second-level HTLC transactions and sweep transactions that recover funds after a channel close.

---

### SignLocalHtlcTx — Sign Our HTLC Transaction (CLN)

| | |
|---|---|
| **ID** | 16 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | To sign an HTLC-success or HTLC-timeout transaction spending from our commitment |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment the HTLC belongs to |
| `tx` | WithSize\<Transaction\> | HTLC transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `wscript` | Octets | Witness script of the HTLC output being spent |
| `option_anchors` | bool | Anchor outputs enabled |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature on the HTLC transaction |

---

### SignLocalHtlcTx2 — Sign Our HTLC Transaction (LDK)

| | |
|---|---|
| **ID** | 20 |
| **Direction** | Node → Signer |
| **Compatibility** | LDK only |
| **When sent** | Same purpose, with explicit HTLC details |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | HTLC transaction |
| `input` | u32 | Input index to sign |
| `per_commitment_number` | u64 | Per-commitment number |
| `offered` | bool | true if this is an offered (outgoing) HTLC |
| `cltv_expiry` | u32 | HTLC CLTV timeout |
| `htlc_amount_msat` | u64 | HTLC amount in millisatoshis |
| `payment_hash` | Sha256 | HTLC payment hash |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Our signature |

---

### SignRemoteHtlcToUs — Sweep Counterparty's HTLC to Us (CLN)

| | |
|---|---|
| **ID** | 13 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | After counterparty force-closes, to sweep an HTLC output that rightfully belongs to us (e.g., we know the preimage) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `remote_per_commitment_point` | PubKey | Counterparty's per-commitment point |
| `tx` | WithSize\<Transaction\> | Sweep transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `wscript` | Octets | Witness script |
| `option_anchors` | bool | Anchor outputs enabled |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Signature |

---

### SignRemoteHtlcTx — Sign Counterparty HTLC Sweep

| | |
|---|---|
| **ID** | 20 |
| **Direction** | Node → Signer |
| **Compatibility** | Both CLN and LDK |
| **When sent** | To spend a counterparty's HTLC output (general case) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | Sweep transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT with metadata |
| `wscript` | Octets | Witness script |
| `remote_per_commitment_point` | PubKey | Counterparty's per-commitment point |
| `option_anchors` | bool | Anchor outputs enabled |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Signature |

---

### SignDelayedPaymentToUs — Sweep Delayed Output (CLN)

| | |
|---|---|
| **ID** | 12 |
| **Direction** | Node → Signer |
| **Compatibility** | CLN only |
| **When sent** | After our commitment is broadcast and CSV delay expires, to sweep the to_local output back to our wallet |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number |
| `tx` | WithSize\<Transaction\> | Sweep transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Signature |

---

### SignPenaltyToUs — Sign Penalty Transaction

| | |
|---|---|
| **ID** | 14 |
| **Direction** | Node → Signer |
| **Compatibility** | Both |
| **When sent** | When counterparty broadcasts a revoked commitment — this signs the justice/penalty transaction that claims all their funds |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `revocation_secret` | DisclosedSecret | The counterparty's revocation secret for the breached state |
| `tx` | WithSize\<Transaction\> | Penalty transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Signature |

---

## 9. CLN Any-Channel Dispatch

These are CLN-specific variants of signing messages dispatched by `lightningd` (the main daemon) rather than the per-channel `channeld` subprocess. They include explicit peer_id/dbid fields so the signer can identify which channel is involved.

---

### SignAnyChannelAnnouncement (ID: 4)

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `announcement` | Octets | Channel announcement bytes |
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: SignAnyChannelAnnouncementReply (ID: 104)**

| Field | Type | Description |
|-------|------|-------------|
| `node_signature` | Signature | Node signature on announcement |
| `bitcoin_signature` | Signature | Bitcoin (funding key) signature |

---

### SignAnyDelayedPaymentToUs (ID: 142)

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number |
| `tx` | WithSize\<Transaction\> | Sweep transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |
| `input` | u32 | Input index |
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: SignTxReply (ID: 112)**

---

### SignAnyRemoteHtlcToUs (ID: 143)

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `remote_per_commitment_point` | PubKey | Counterparty's per-commitment point |
| `tx` | WithSize\<Transaction\> | Sweep transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |
| `option_anchors` | bool | Anchor outputs enabled |
| `input` | u32 | Input index |
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: SignTxReply (ID: 112)**

---

### SignAnyPenaltyToUs (ID: 144)

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `revocation_secret` | DisclosedSecret | Revocation secret |
| `tx` | WithSize\<Transaction\> | Penalty transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |
| `input` | u32 | Input index |
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: SignTxReply (ID: 112)**

---

### SignAnyLocalHtlcTx (ID: 146)

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `commitment_number` | u64 | Commitment number |
| `tx` | WithSize\<Transaction\> | HTLC transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `wscript` | Octets | Witness script |
| `option_anchors` | bool | Anchor outputs enabled |
| `input` | u32 | Input index |
| `peer_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: SignTxReply (ID: 112)**

---

## 10. Gossip & Announcements

---

### SignChannelAnnouncement (ID: 2)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a channel announcement for gossip broadcast |

**Request:** `announcement: Octets`

**Response: SignChannelAnnouncementReply (ID: 102)**

| Field | Type | Description |
|-------|------|-------------|
| `node_signature` | Signature | Signed with node key |
| `bitcoin_signature` | Signature | Signed with funding key |

---

### SignChannelUpdate (ID: 3)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a channel update (fee changes, enable/disable) |

**Request:** `update: Octets`

**Response: SignChannelUpdateReply (ID: 103)**

| Field | Type | Description |
|-------|------|-------------|
| `update` | Octets | Signed channel update bytes |

---

### SignNodeAnnouncement (ID: 6)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign node announcement for gossip |

**Request:** `announcement: Octets`

**Response: SignNodeAnnouncementReply (ID: 106)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | Signature | Node announcement signature |

---

### SignGossipMessage (ID: 1006) — LDK Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign an LDK gossip message |

**Request:** `message: Octets`

**Response: SignGossipMessageReply (ID: 1106)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | Signature | Message signature |

---

## 11. Invoice Signing

---

### SignInvoice (ID: 8)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a BOLT11 invoice |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `u5bytes` | Octets | Base32-encoded invoice data |
| `hrp` | Octets | Human-readable part ("lnbc", "lntb", etc.) |

**Response: SignInvoiceReply (ID: 108)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | RecoverableSignature | 65-byte recoverable ECDSA signature |

---

### PreapproveInvoice (ID: 38)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | Before paying an invoice, to check if the signer approves the payment |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `invstring` | WireString | Full BOLT11 invoice string |

**Response: PreapproveInvoiceReply (ID: 138)**

| Field | Type | Description |
|-------|------|-------------|
| `result` | bool | true = approved, false = rejected |

**What the signer checks:** Invoice expiry, amount limits, velocity limits, destination allowlist.

---

### PreapproveKeysend (ID: 39)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | Before sending a keysend (spontaneous) payment |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `destination` | PubKey | Destination node |
| `payment_hash` | Sha256 | Payment hash |
| `amount_msat` | u64 | Amount in millisatoshis |

**Response: PreapproveKeysendReply (ID: 139)**

| Field | Type | Description |
|-------|------|-------------|
| `result` | bool | true = approved |

---

## 12. BOLT12 Signing

---

### SignBolt12 (ID: 25)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a BOLT12 offer, invoice request, or invoice |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `message_name` | WireString | Message type ("offer", "invoice_request", "invoice") |
| `field_name` | WireString | Field being signed ("signature") |
| `merkle_root` | Sha256 | Merkle root of the TLV fields |
| `public_tweak` | Octets | Optional key tweak |

**Response: SignBolt12Reply (ID: 125)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | Signature | Schnorr signature |

---

### SignBolt12V2 (ID: 41)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | Enhanced BOLT12 signing with additional message context |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `message_name` | WireString | Message type |
| `field_name` | WireString | Field being signed |
| `merkle_root` | Sha256 | Merkle root |
| `info` | Octets | Full message bytes for inspection |
| `public_tweak` | Octets | Optional key tweak |

**Response: SignBolt12V2Reply (ID: 141)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | Signature | Schnorr signature |

---

## 13. On-chain / PSBT Signing

---

### SignWithdrawal (ID: 7)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign an on-chain transaction (wallet withdrawal, channel funding) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `utxos` | Array\<Utxo\> | UTXOs being spent |
| `psbt` | WithSize\<StreamedPSBT\> | Unsigned PSBT |

**Response: SignWithdrawalReply (ID: 107)**

| Field | Type | Description |
|-------|------|-------------|
| `psbt` | WithSize\<PsbtWrapper\> | Signed PSBT |

**What the signer validates:** All policy-onchain-* rules (fee range, unknown outputs, non-malleable inputs, max size, scriptPubKey validity, funding match, etc.)

---

### SignAnchorspend (ID: 147) — CLN Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a CPFP transaction spending a channel anchor output for fee bumping |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `peer_id` | PubKey | Channel counterparty |
| `dbid` | u64 | Channel database ID |
| `utxos` | Array\<Utxo\> | Wallet UTXOs for fee funding |
| `psbt` | WithSize\<StreamedPSBT\> | Unsigned PSBT |

**Response: SignAnchorspendReply (ID: 148)**

| Field | Type | Description |
|-------|------|-------------|
| `psbt` | WithSize\<PsbtWrapper\> | Signed PSBT |

---

### SignHtlcTxMingle (ID: 149) — CLN Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a PSBT that mixes HTLC outputs (splicing/coinjoin context) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `peer_id` | PubKey | Counterparty |
| `dbid` | u64 | Channel database ID |
| `utxos` | Array\<Utxo\> | UTXOs being spent |
| `psbt` | WithSize\<StreamedPSBT\> | Unsigned PSBT |

**Response: SignHtlcTxMingleReply (ID: 150)**

| Field | Type | Description |
|-------|------|-------------|
| `psbt` | WithSize\<PsbtWrapper\> | Signed PSBT |

---

### SignSpliceTx (ID: 29) — CLN Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign a splice transaction (adding/removing funds from a live channel) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tx` | WithSize\<Transaction\> | Splice transaction |
| `psbt` | WithSize\<PsbtWrapper\> | PSBT |
| `remote_funding_key` | PubKey | Counterparty's funding key |
| `input_index` | u32 | Input index to sign |

**Response: SignTxReply (ID: 112)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | BitcoinSignature | Signature |

---

## 14. Key Derivation & Verification

---

### GetChannelBasepoints (ID: 10)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To retrieve a channel's basepoints and funding key for commitment construction |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `node_id` | PubKey | Counterparty node ID |
| `dbid` | u64 | Channel database ID |

**Response: GetChannelBasepointsReply (ID: 110)**

| Field | Type | Description |
|-------|------|-------------|
| `basepoints` | Basepoints | revocation, payment, htlc, delayed_payment basepoints |
| `funding` | PubKey | Our funding public key for this channel |

---

### DeriveSecret (ID: 27)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To derive a secret for a specific context (e.g., onion encryption) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `info` | Octets | Key derivation context bytes |

**Response: DeriveSecretReply (ID: 127)**

| Field | Type | Description |
|-------|------|-------------|
| `secret` | Secret | Derived 32-byte secret |

---

### CheckPubKey (ID: 28)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To verify a public key matches the signer's derived key at a wallet index |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `index` | u32 | Wallet derivation index |
| `pubkey` | PubKey | Public key to check |

**Response: CheckPubKeyReply (ID: 128)**

| Field | Type | Description |
|-------|------|-------------|
| `ok` | bool | Key matches |

---

## 15. Elliptic Curve Operations

---

### Ecdh (ID: 1)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To compute ECDH shared secret (used for onion routing encryption, peer handshake) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `point` | PubKey | Counterparty's public key |

**Response: EcdhReply (ID: 100)**

| Field | Type | Description |
|-------|------|-------------|
| `secret` | Secret | 32-byte ECDH shared secret |

---

### SignMessage (ID: 23)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To sign an arbitrary message with the node's private key |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `message` | Octets | Message bytes to sign |

**Response: SignMessageReply (ID: 123)**

| Field | Type | Description |
|-------|------|-------------|
| `signature` | RecoverableSignature | 65-byte recoverable signature |

---

## 16. Blockchain Synchronization

These messages keep the signer's chain state synchronized for on-chain validation rules.

---

### TipInfo (ID: 2002)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To query the signer's current view of the blockchain tip |

**Request fields:** None

**Response: TipInfoReply (ID: 2102)**

| Field | Type | Description |
|-------|------|-------------|
| `height` | u32 | Block height |
| `block_hash` | BlockHash | Block hash at tip |

---

### ForwardWatches (ID: 2003)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To get the list of transactions and outpoints the signer wants to monitor going forward |

**Request fields:** None

**Response: ForwardWatchesReply (ID: 2103)**

| Field | Type | Description |
|-------|------|-------------|
| `txids` | Array\<Txid\> | Transactions to watch |
| `outpoints` | Array\<OutPoint\> | Outpoints to watch |

---

### ReverseWatches (ID: 2004)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To get watches that should be removed (no longer relevant) |

**Request fields:** None

**Response: ReverseWatchesReply (ID: 2104)**

| Field | Type | Description |
|-------|------|-------------|
| `txids` | Array\<Txid\> | Transactions to unwatch |
| `outpoints` | Array\<OutPoint\> | Outpoints to unwatch |

---

### AddBlock (ID: 2005)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | When a new block is confirmed, to advance the signer's chain state |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `header` | Octets | 80-byte Bitcoin block header (consensus-encoded) |
| `unspent_proof` | Option\<TxoProof\> | TXOO (Transaction Output Oracle) proof of unspent outputs |

**Response: AddBlockReply (ID: 2105)** — Empty

---

### RemoveBlock (ID: 2006)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | During a blockchain reorganization, to roll back the signer's chain state |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `unspent_proof` | Option\<LargeOctets\> | TXOO proof |
| `prev_block_header` | BlockHeader | Previous block header to revert to |
| `prev_filter_header` | FilterHeader | BIP158 compact filter header |

**Response: RemoveBlockReply (ID: 2106)** — Empty

---

### BlockChunk (ID: 2009)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To stream full block data when the signer needs to inspect transactions (compact filter false positive) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `hash` | BlockHash | Block hash |
| `offset` | u32 | Chunk offset within block data |
| `content` | Octets | Block data chunk |

**Response: BlockChunkReply (ID: 2109)** — Empty

---

### GetHeartbeat (ID: 2008)

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To get a signed heartbeat proving the signer is alive and at a known chain state |

**Request fields:** None

**Response: GetHeartbeatReply (ID: 2108)**

| Field | Type | Description |
|-------|------|-------------|
| `heartbeat` | Octets | Serialized and signed heartbeat |

---

## 17. Miscellaneous

---

### ClientHsmFd (ID: 9) — CLN Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | To notify the signer of a new client connection (legacy CLN file-descriptor passing) |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `peer_id` | PubKey | Peer node ID |
| `dbid` | u64 | Channel database ID |
| `capabilities` | u64 | Capability flags (unused) |

**Response: ClientHsmFdReply (ID: 109)** — Empty

---

### GetSecureRandomBytes (ID: 1036) — LDK Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | When the node needs cryptographically secure random bytes from the signer's RNG |

**Request fields:** None

**Response: GetSecureRandomBytesReply (ID: 1136)**

| Field | Type | Description |
|-------|------|-------------|
| `random_bytes` | Secret | 32 cryptographically secure random bytes |

---

### Ping (ID: 1000)

| | |
|---|---|
| **Direction** | Bidirectional |
| **When sent** | Keep-alive / latency measurement |

**Request fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | u16 | Ping identifier |
| `message` | WireString | Arbitrary message |

**Response: Pong (ID: 1100)**

| Field | Type | Description |
|-------|------|-------------|
| `id` | u16 | Echoed ping ID |
| `message` | WireString | Echoed message |

---

### Memleak (ID: 33) — CLN Only

| | |
|---|---|
| **Direction** | Node → Signer |
| **When sent** | Developer/debug memory leak detection |

**Request fields:** None

**Response: MemleakReply (ID: 133)**

| Field | Type | Description |
|-------|------|-------------|
| `result` | bool | true if memory leak detected |

---

## 18. Error Handling

---

### SignerError (ID: 3000)

| | |
|---|---|
| **Direction** | Signer → Node (unsolicited) |
| **When sent** | When the signer encounters a fatal error it needs to communicate to the node |

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `code` | u16 | Error code |
| `message` | WireString | Human-readable error description |

**Known error codes:**

| Code | Constant | Description |
|------|----------|-------------|
| 401 | CODE_ORPHAN_BLOCK | Received orphan block during chain sync |

---

## 19. Protocol Flows

### Channel Establishment

```
Node                                    Signer
 │                                        │
 │─── HsmdInit / HsmdInit2 ─────────────►│  Initialize session
 │◄── HsmdInitReplyV4 / HsmdInit2Reply ──│  Return node_id, keys
 │                                        │
 │─── NewChannel(peer, dbid) ────────────►│  Register channel
 │◄── NewChannelReply ───────────────────│
 │                                        │
 │─── GetChannelBasepoints ──────────────►│  Get our channel keys
 │◄── GetChannelBasepointsReply ─────────│
 │                                        │
 │─── GetPerCommitmentPoint(0) ──────────►│  Get first commitment point
 │◄── GetPerCommitmentPointReply ────────│
 │                                        │
 │  ... negotiate with peer ...           │
 │                                        │
 │─── SetupChannel(params) ─────────────►│  Configure channel
 │◄── SetupChannelReply ────────────────│
 │                                        │
 │─── SignRemoteCommitmentTx(0) ────────►│  Sign peer's initial commitment
 │◄── SignTxReply(sig) ─────────────────│
 │                                        │
 │─── ValidateCommitmentTx(0, peer_sig) ►│  Validate our initial commitment
 │◄── ValidateCommitmentTxReply ────────│
 │                                        │
 │─── SignWithdrawal(funding_psbt) ─────►│  Sign funding transaction
 │◄── SignWithdrawalReply(signed_psbt) ──│
 │                                        │
 │  ... funding tx confirms ...           │
 │                                        │
 │─── CheckOutpoint ────────────────────►│  Verify funding confirmed
 │◄── CheckOutpointReply(is_buried) ────│
 │                                        │
 │─── LockOutpoint ─────────────────────►│  Lock funding UTXO
 │◄── LockOutpointReply ───────────────│
 │                                        │
```

### Normal Operation — Commitment Update

```
Sending commitment_signed to peer:

Node                                    Signer
 │                                        │
 │─── SignRemoteCommitmentTx(n+1) ──────►│  Sign peer's new commitment
 │◄── SignTxReply(sig) ─────────────────│
 │                                        │
 │  ── send commitment_signed to peer ──► │
 │                                        │


Receiving commitment_signed from peer:

Node                                    Signer
 │                                        │
 │  ◄── receive commitment_signed ─────── │
 │                                        │
 │─── ValidateCommitmentTx(n+1, sig) ──►│  Validate our new commitment
 │◄── ValidateCommitmentTxReply ────────│
 │                                        │
 │─── RevokeCommitmentTx(n) ───────────►│  Revoke old commitment (v5+)
 │◄── RevokeCommitmentTxReply(secret) ──│
 │                                        │
 │  ── send revoke_and_ack to peer ─────► │
 │                                        │


Receiving revoke_and_ack from peer:

Node                                    Signer
 │                                        │
 │  ◄── receive revoke_and_ack ────────── │
 │                                        │
 │─── ValidateRevocation(m, secret) ────►│  Validate peer's revocation
 │◄── ValidateRevocationReply ──────────│
 │                                        │
```

### Cooperative Close

```
Node                                    Signer
 │                                        │
 │  ... negotiate closing_signed ...      │
 │                                        │
 │─── SignMutualCloseTx(close_tx) ──────►│  Sign close transaction
 │◄── SignTxReply(sig) ─────────────────│  (validates destination,
 │                                        │   value, fee, no HTLCs)
 │  ── broadcast close tx ──────────────► │
 │                                        │
 │─── ForgetChannel ────────────────────►│  Clean up signer state
 │◄── ForgetChannelReply ──────────────│
 │                                        │
```

### Force Close & Sweep

```
Node                                    Signer
 │                                        │
 │  ... our commitment is broadcast ...   │
 │  ... CSV delay expires ...             │
 │                                        │
 │─── SignDelayedPaymentToUs(sweep) ────►│  Sweep to_local output
 │◄── SignTxReply(sig) ─────────────────│
 │                                        │
 │─── SignLocalHtlcTx(htlc_timeout) ───►│  Sign HTLC timeout tx
 │◄── SignTxReply(sig) ─────────────────│
 │                                        │
 │─── SignLocalHtlcTx(htlc_success) ───►│  Sign HTLC success tx
 │◄── SignTxReply(sig) ─────────────────│
 │                                        │
```

### Penalty (Justice) Transaction

```
Node                                    Signer
 │                                        │
 │  ... detect revoked commitment ...     │
 │                                        │
 │─── SignPenaltyToUs(rev_secret, tx) ──►│  Sign penalty tx
 │◄── SignTxReply(sig) ─────────────────│  (claims ALL funds)
 │                                        │
 │  ── broadcast penalty tx ────────────► │
 │                                        │
```

---

## Message ID Reference Table

| ID | Message | Direction | Compatibility |
|----|---------|-----------|---------------|
| 1 | Ecdh | Request | Both |
| 2 | SignChannelAnnouncement | Request | Both |
| 3 | SignChannelUpdate | Request | Both |
| 4 | SignAnyChannelAnnouncement | Request | CLN |
| 5 | SignCommitmentTx | Request | CLN |
| 6 | SignNodeAnnouncement | Request | Both |
| 7 | SignWithdrawal | Request | Both |
| 8 | SignInvoice | Request | Both |
| 9 | ClientHsmFd | Request | CLN |
| 10 | GetChannelBasepoints | Request | Both |
| 11 | HsmdInit | Request | CLN |
| 12 | SignDelayedPaymentToUs | Request | CLN |
| 13 | SignRemoteHtlcToUs | Request | CLN |
| 14 | SignPenaltyToUs | Request | Both |
| 16 | SignLocalHtlcTx | Request | CLN |
| 18 | GetPerCommitmentPoint | Request | CLN |
| 19 | SignRemoteCommitmentTx | Request | CLN |
| 20 | SignRemoteHtlcTx / SignLocalHtlcTx2 | Request | Both |
| 21 | SignMutualCloseTx | Request | CLN |
| 22 | CheckFutureSecret | Request | Both |
| 23 | SignMessage | Request | Both |
| 25 | SignBolt12 | Request | Both |
| 27 | DeriveSecret | Request | Both |
| 28 | CheckPubKey | Request | Both |
| 29 | SignSpliceTx | Request | CLN |
| 30 | NewChannel | Request | CLN |
| 31 | SetupChannel | Request | Both |
| 32 | CheckOutpoint | Request | Both |
| 33 | Memleak | Request | CLN |
| 34 | ForgetChannel | Request | Both |
| 35 | ValidateCommitmentTx | Request | CLN |
| 36 | ValidateRevocation | Request | Both |
| 37 | LockOutpoint | Request | Both |
| 38 | PreapproveInvoice | Request | Both |
| 39 | PreapproveKeysend | Request | Both |
| 40 | RevokeCommitmentTx | Request | CLN (v5+) |
| 41 | SignBolt12V2 | Request | Both |
| 90 | HsmdDevPreinit | Request | Dev |
| 99 | HsmdDevPreinit2 | Request | Dev |
| 100 | EcdhReply | Response | Both |
| 102 | SignChannelAnnouncementReply | Response | Both |
| 103 | SignChannelUpdateReply | Response | Both |
| 104 | SignAnyChannelAnnouncementReply | Response | CLN |
| 105 | SignCommitmentTxReply | Response | Both |
| 106 | SignNodeAnnouncementReply | Response | Both |
| 107 | SignWithdrawalReply | Response | Both |
| 108 | SignInvoiceReply | Response | Both |
| 109 | ClientHsmFdReply | Response | CLN |
| 110 | GetChannelBasepointsReply | Response | Both |
| 112 | SignTxReply | Response | Both |
| 113 | HsmdInitReplyV2 | Response | CLN |
| 114 | HsmdInitReplyV4 | Response | CLN |
| 118 | GetPerCommitmentPointReply | Response | CLN |
| 122 | CheckFutureSecretReply | Response | Both |
| 123 | SignMessageReply | Response | Both |
| 125 | SignBolt12Reply | Response | Both |
| 127 | DeriveSecretReply | Response | Both |
| 128 | CheckPubKeyReply | Response | Both |
| 130 | NewChannelReply | Response | CLN |
| 131 | SetupChannelReply | Response | Both |
| 132 | CheckOutpointReply | Response | Both |
| 133 | MemleakReply | Response | CLN |
| 134 | ForgetChannelReply | Response | Both |
| 135 | ValidateCommitmentTxReply | Response | Both |
| 136 | ValidateRevocationReply | Response | Both |
| 137 | LockOutpointReply | Response | Both |
| 138 | PreapproveInvoiceReply | Response | Both |
| 139 | PreapproveKeysendReply | Response | Both |
| 140 | RevokeCommitmentTxReply | Response | CLN (v5+) |
| 141 | SignBolt12V2Reply | Response | Both |
| 142 | SignAnyDelayedPaymentToUs | Request | CLN |
| 143 | SignAnyRemoteHtlcToUs | Request | CLN |
| 144 | SignAnyPenaltyToUs | Request | CLN |
| 146 | SignAnyLocalHtlcTx | Request | CLN |
| 147 | SignAnchorspend | Request | CLN |
| 148 | SignAnchorspendReply | Response | CLN |
| 149 | SignHtlcTxMingle | Request | CLN |
| 150 | SignHtlcTxMingleReply | Response | CLN |
| 190 | HsmdDevPreinitReply | Response | Dev |
| 1000 | Ping | Request | Both |
| 1005 | SignLocalCommitmentTx2 | Request | LDK |
| 1006 | SignGossipMessage | Request | LDK |
| 1011 | HsmdInit2 | Request | LDK |
| 1012 | NodeInfo | Request | LDK |
| 1018 | GetPerCommitmentPoint2 | Request | LDK |
| 1019 | SignRemoteCommitmentTx2 | Request | LDK |
| 1021 | SignMutualCloseTx2 | Request | LDK |
| 1035 | ValidateCommitmentTx2 | Request | LDK |
| 1036 | GetSecureRandomBytes | Request | LDK |
| 1100 | Pong | Response | Both |
| 1106 | SignGossipMessageReply | Response | LDK |
| 1111 | HsmdInit2Reply | Response | LDK |
| 1112 | NodeInfoReply | Response | LDK |
| 1118 | GetPerCommitmentPoint2Reply | Response | LDK |
| 1119 | SignCommitmentTxWithHtlcsReply | Response | LDK |
| 1136 | GetSecureRandomBytesReply | Response | LDK |
| 2002 | TipInfo | Request | Both |
| 2003 | ForwardWatches | Request | Both |
| 2004 | ReverseWatches | Request | Both |
| 2005 | AddBlock | Request | Both |
| 2006 | RemoveBlock | Request | Both |
| 2008 | GetHeartbeat | Request | Both |
| 2009 | BlockChunk | Request | Both |
| 2102 | TipInfoReply | Response | Both |
| 2103 | ForwardWatchesReply | Response | Both |
| 2104 | ReverseWatchesReply | Response | Both |
| 2105 | AddBlockReply | Response | Both |
| 2106 | RemoveBlockReply | Response | Both |
| 2108 | GetHeartbeatReply | Response | Both |
| 2109 | BlockChunkReply | Response | Both |
| 3000 | SignerError | Error | Signer→Node |

---

*Source: [VLS Repository](https://gitlab.com/lightning-signer/validating-lightning-signer), `vls-protocol/src/msgs.rs` and `vls-protocol-signer/src/handler.rs`*
