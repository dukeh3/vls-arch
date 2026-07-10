# Open Channel — Design

Compound V2 proxy messages for the LDK-based VLS proxy protocol.

Entities: **Signer** (VLS core), **SP** (signer-proxy / dispatch),
**NP** (node-proxy / client), **Alice** or **Bob** (LDK node).

---

## Alice (A01–A04) — Funder

5 round-trips over the slow link (NP ↔ SP):
4 channel-protocol + 1 wallet (SignWithdrawal).

```mermaid
sequenceDiagram
    participant Signer as Signer
    participant SP as SP
    participant NP as NP
    participant Alice as Alice
    participant Bob as Bob

    Note over Alice,Bob: === A01: Create channel ===

    Alice->>NP: derive_channel_signer(channel_keys_id)

    NP->>SP: vls_create_channel(channel_keys_id)
    SP->>Signer: NewChannel
    Signer-->>SP: OK
    SP->>Signer: GetChannelBasepoints
    Signer-->>SP: Af, {Ar,Ap,Ad,Ah}
    SP->>Signer: GetPerCommitmentPoint(0)
    Signer-->>SP: ap0
    SP->>Signer: GetPerCommitmentPoint(1)
    Signer-->>SP: ap1
    SP-->>NP: Af, {Ar,Ap,Ad,Ah}, ap0, ap1

    Note over NP: Cache: Af, basepoints, ap0, ap1

    NP-->>Alice: DynSigner
    Alice->>NP: get_per_commitment_point(0)
    NP-->>Alice: ap0 (cache)
    Alice->>NP: get_per_commitment_point(1)
    NP-->>Alice: ap1 (cache)

    Alice->>Bob: open_channel(Af, basepoints, ap0)
    Bob->>Alice: accept_channel(Bf, basepoints, bp0)

    Note over Alice,Bob: === A02: Sign counterparty commitment ===

    Alice->>NP: sign_counterparty_commitment(channel_params(Bf, bp_B, funding), commit_0(bp0))

    NP->>SP: vls_funding_created(Bf, {Br,Bp,Bd,Bh}, funding_outpoint, bp0)
    SP->>Signer: SetupChannel(Bf, {Br,Bp,Bd,Bh}, funding_outpoint)
    Signer-->>SP: OK
    SP->>Signer: SignRemoteCommitmentTx2(bp0, commit=0)
    Signer-->>SP: sig_Af
    SP-->>NP: sig_Af

    NP-->>Alice: sig_Af
    Alice->>Bob: funding_created(sig_Af)
    Bob->>Alice: funding_signed(sig_Bf)

    Note over Alice,Bob: === A03: Confirm counterparty signature ===

    Alice->>NP: validate_holder_commitment(commit=0, sig_Bf)

    NP->>SP: vls_confirm_counterparty_sig(commit=0, sig_Bf)
    SP->>Signer: ValidateCommitmentTx2(commit=0, sig_Bf)
    Signer-->>SP: next_pcp
    SP->>Signer: GetPerCommitmentPoint(2)
    Signer-->>SP: ap2
    SP-->>NP: ap2

    Note over NP: Cache: PCP(2)=ap2

    NP-->>Alice: validated

    Alice->>NP: get_per_commitment_point(2)
    NP-->>Alice: ap2 (cache)

    Note over Alice: can_advance()=true

    Note over Alice,Bob: === vls_sign_withdrawal (wallet) ===

    Alice->>NP: sign_transaction(psbt) [VlsBdkSigner]

    NP->>SP: vls_sign_withdrawal(utxos, psbt)
    SP->>Signer: SignWithdrawal(utxos, psbt)
    Signer-->>SP: signed_psbt
    SP-->>NP: signed_psbt

    NP-->>Alice: signed_psbt
    Alice->>Alice: broadcast funding tx

    Note over Alice,Bob: === A04: Channel ready (after funding confirms) ===

    Note over Alice: Funding confirms
    Alice->>Bob: channel_ready(ap1)

    Alice->>NP: channel_ready(channel_keys_id)

    NP->>SP: vls_channel_ready(channel_keys_id)
    SP->>Signer: CheckOutpoint
    Signer-->>SP: is_buried=true
    SP->>Signer: LockOutpoint
    Signer-->>SP: OK
    SP-->>NP: is_buried

    NP-->>Alice: is_buried
```

### RTT Summary — Alice

| Step | Compound message | VLS calls inside | RTTs |
|------|-----------------|------------------|------|
| A01 | `vls_create_channel` | NewChannel + GetBasepoints + PCP(0) + PCP(1) | 1 |
| A02 | `vls_funding_created` | SetupChannel + SignRemoteCommitmentTx2 | 1 |
| A03 | `vls_confirm_counterparty_sig` | ValidateCommitmentTx2 + PCP(2) | 1 |
| — | `vls_sign_withdrawal` | SignWithdrawal via `VlsBdkSigner` | 1 |
| A04 | `vls_channel_ready` | CheckOutpoint + LockOutpoint | 1 |
| | | **Total** | **5** |

---

## Bob (B01–B03) — Acceptor

3 round-trips over the slow link (NP ↔ SP).

```mermaid
sequenceDiagram
    participant Signer as Signer
    participant SP as SP
    participant NP as NP
    participant Bob as Bob
    participant Alice as Alice

    Note over Bob,Alice: === B01: Create channel ===

    Alice->>Bob: open_channel(Af, basepoints, ap0)

    Bob->>NP: derive_channel_signer(channel_keys_id)

    NP->>SP: vls_create_channel(channel_keys_id)
    SP->>Signer: NewChannel
    Signer-->>SP: OK
    SP->>Signer: GetChannelBasepoints
    Signer-->>SP: Bf, {Br,Bp,Bd,Bh}
    SP->>Signer: GetPerCommitmentPoint(0)
    Signer-->>SP: bp0
    SP->>Signer: GetPerCommitmentPoint(1)
    Signer-->>SP: bp1
    SP-->>NP: Bf, {Br,Bp,Bd,Bh}, bp0, bp1

    Note over NP: Cache: Bf, basepoints, bp0, bp1

    NP-->>Bob: DynSigner
    Bob->>NP: get_per_commitment_point(0)
    NP-->>Bob: bp0 (cache)
    Bob->>NP: get_per_commitment_point(1)
    NP-->>Bob: bp1 (cache)

    Bob->>Alice: accept_channel(Bf, basepoints, bp0)
    Alice->>Bob: funding_created(sig_Af)

    Note over Bob,Alice: === B02: Confirm counterparty signature + sign ===

    Bob->>NP: validate_holder_commitment(commit=0, sig_Af)
    Note over NP: Pending validate
    NP-->>Bob: (pending)

    Bob->>NP: sign_counterparty_commitment(channel_params(Af, bp_A, funding), commit_0(ap0))

    NP->>SP: vls_funding_signed(Af, {Ar,Ap,Ad,Ah}, funding_outpoint, sig_Af, ap0)
    SP->>Signer: SetupChannel(Af, {Ar,Ap,Ad,Ah}, funding_outpoint)
    Signer-->>SP: OK
    SP->>Signer: ValidateCommitmentTx2(commit=0, sig_Af)
    Signer-->>SP: next_pcp
    SP->>Signer: SignRemoteCommitmentTx2(ap0, commit=0)
    Signer-->>SP: sig_Bf
    SP->>Signer: GetPerCommitmentPoint(2)
    Signer-->>SP: bp2
    SP-->>NP: sig_Bf, bp2

    Note over NP: Cache: PCP(2)=bp2

    NP-->>Bob: sig_Bf

    Bob->>NP: get_per_commitment_point(2)
    NP-->>Bob: bp2 (cache)
    Note over Bob: can_advance()=true

    Bob->>Alice: funding_signed(sig_Bf)

    Note over Bob,Alice: === B03: Channel ready (after funding confirms) ===

    Note over Bob: Funding confirms
    Bob->>Alice: channel_ready(bp1)

    Bob->>NP: channel_ready(channel_keys_id)

    NP->>SP: vls_channel_ready(channel_keys_id)
    SP->>Signer: CheckOutpoint
    Signer-->>SP: is_buried=true
    SP->>Signer: LockOutpoint
    Signer-->>SP: OK
    SP-->>NP: is_buried

    NP-->>Bob: is_buried
```

### RTT Summary — Bob

| Step | Compound message | VLS calls inside | RTTs |
|------|-----------------|------------------|------|
| B01 | `vls_create_channel` | NewChannel + GetBasepoints + PCP(0) + PCP(1) | 1 |
| B02 | `vls_funding_signed` | SetupChannel + ValidateCommitmentTx2 + SignRemoteCommitmentTx2 + PCP(2) | 1 |
| B03 | `vls_channel_ready` | CheckOutpoint + LockOutpoint | 1 |
| | | **Total** | **3** |
