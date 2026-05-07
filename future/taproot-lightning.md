# Taproot Lightning (Future)

Placeholder for taproot-based Lightning channel constructions (e.g., simple taproot channels, PTLCs).

This will map the same high-level reference notation (A+B style from the SVG) to taproot-native key structures — MuSig2 funding outputs, tapscript commitment outputs, point-time-locked contracts instead of HTLCs, etc.

## Status

Not started. Depends on finalizing the terminology bridge between reference notation and today's BOLT 3 implementation first.

## Open Questions

- MuSig2 for funding output (replaces 2-of-2 multisig)
- Tapscript branches for revocation / delayed paths (replaces script-based OP_IF)
- PTLCs vs HTLCs (adaptor signatures vs hash preimages)
- How VLS signer API surface changes (new calls? modified calls?)
