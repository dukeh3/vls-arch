# VLS Architecture Docs

Documentation for how Lightning protocol messages map to VLS signer API calls.

## Structure

```
reference/          — protocol-level reference material (no VLS)
  lightning-protocol.md         How Lightning works (update-commit-revoke, HTLCs, lifecycle)
  lightning-key-derivation.md   Notation legend + key derivation: SVG → BOLT 3 formulas
  scenarios.md                  Scenario roadmap (happy path + adversarial)
  01-alice-pays-bob.md          Lightning messages for scenario 01
current/            — VLS signer calls as they work TODAY (separate API calls)
  01-alice-pays-bob.md
future/             — optimized/batched VLS calls + taproot exploration
  01-alice-pays-bob.md
  taproot-lightning.md          placeholder
```

## Terminology Layers

1. **High-level reference** (drawio SVG) — simplified notation like `A + b1 | B + dt` capturing the function of each spending path
2. **Today's Lightning** (BOLT 3) — EC point derivation with basepoints and per-commitment points: `rp(Ar, bp1) | dp(Bd, bp1) + dt`
3. **Future taproot Lightning** — same functional roles, taproot-native constructions (MuSig2, tapscript, PTLCs)

The key derivation doc bridges layer 1 → 2. The taproot doc will bridge 1 → 3.

## Scenarios

See [reference/scenarios.md](reference/scenarios.md) for the full roadmap (happy path 01–999, adversarial 1000+).

Currently documented:

| # | Name | Status |
|---|------|--------|
| 01 | Alice Pays Bob | reference, current, future |

## Current Focus

- Ensure terminology is consistent across all three layers
- Each scenario has the same file name in reference/, current/, and future/
- reference/ describes *what* happens on the wire
- current/ describes *what the signer does* today (one call at a time)
- future/ describes *what the signer could do* with batching or taproot

## Epics

| # | Epic | Lightning | Signer | Description |
|---|------|-----------|--------|-------------|
| 0 | Today's VLS | BOLT 3 (script-based) | Single signer, separate calls | How VLS works right now — one API call per operation, no batching |
| 1 | Optimised VLS | BOLT 3 (script-based) | Single signer, batched calls | Reduce round-trips by batching calls that don't depend on each other |
| 2 | Multi-signer VLS | BOLT 3 (script-based) | Multi-signer (quorum/threshold) | Distribute signing across multiple signers on today's Lightning |
| 3 | Taproot multi-signer | Taproot (MuSig2/tapscript) | Multi-signer | Multi-signer on taproot Lightning — MuSig2 funding, PTLCs, tapscript branches |

### Epic 0 — Today's VLS
Status: **in progress** (`current/`)

Document the VLS signer API as it works today. Each call is separate. Establishes the baseline terminology and call patterns that all future epics build on.

### Epic 1 — Optimised VLS (single signer, batched)
Status: **in progress** (`future/01-alice-pays-bob.md`)

Same single-signer model, but batch calls where sequencing allows. E.g., NewChannel + GetChannelBasepoints + GetPerCommitmentPoint in one round-trip. Identifies which calls have real data dependencies and which are artificially sequential today.

### Epic 2 — Multi-signer VLS on Lightning 1.0
Status: **not started**

Distribute the signing role across a quorum of signers (e.g., 2-of-3 threshold). Still script-based BOLT 3 channels. Key questions: how does threshold signing work with the 2-of-2 funding multisig? Do we need MPC for the funding key, or can we restructure?

### Epic 3 — Multi-signer VLS on Taproot Lightning
Status: **not started** (`future/taproot-lightning.md`)

Taproot-native channels with MuSig2 funding outputs and tapscript commitment branches. Multi-signer maps more naturally here — MuSig2 already supports threshold/quorum constructions. PTLCs replace HTLCs (adaptor signatures instead of hash preimages).
