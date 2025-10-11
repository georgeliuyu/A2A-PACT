Contracts (Reference Implementation)

Scope
- Minimal on-chain core: escrow accounting, timers, symmetric forfeiture, events, pull-based withdrawals.

Must-have
- Subject resolution for direct/2771/4337 calls
- Invariants: zero-fee conservation; single credit; timeout precedence
- EIP-712 settlements with per-signer nonces and deadlines; IERC1271 support
- Withdrawals: nonReentrant; safe ERC-20 transfers; native/erc-20 support

Nice-to-have
- Batch ops; metadata hooks; role-gated top-ups

Tooling
- Foundry or Hardhat for compilation/tests. Keep configs minimal and reproducible.

