# Minimal End‑to‑End Flow (NESP)

- Create order → Deposit escrow → Accept → Mark ready → Approve receipt → Withdraw
- Dispute path: Raise dispute → settleWithSigs (EIP‑712) → Withdraw
- Timeout path: Raise dispute → timeoutForfeit

TypedData sample
- See `EXAMPLES/settlement_typed_data.json`
- Original JSON (if not moved) at `examples/settlement_typed_data.json`

Events
- OrderCreated, EscrowDeposited, Accepted, DisputeRaised, AmountSettled/Settled, Forfeited, Balance{Credited,Withdrawn}

Note: This is a documentation placeholder; no executable code.

