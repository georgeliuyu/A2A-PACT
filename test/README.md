Test Plan

Focus on invariants and state machine:
- Zero-fee conservation; single credit; pull settlement
- Timeout precedence; bounds `0 ≤ A ≤ E`
- Subject guards for 2771/4337 and direct calls
- EIP-712/1271 signature validation and replay protection
- Withdrawals idempotence and reentrancy protection

Suggested Cases (map to `nesp_erc.md` §10)
1. Straight-through settlement
2. Signature settlement (dispute + EIP-712)
3. Symmetric forfeiture (timeout)
4. Third-party gifting (ETH/ERC-20)
5. Subject guards (2771/4337)
6. Timeout precedence
7. Zero-fee invariant
8. Withdrawal idempotence

