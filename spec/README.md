NESP Spec Index

- SSOT (canonical): `docs/zh/nesp_spec.md`
- ERC-facing draft (English): `nesp_erc.md`

Rules
- Any semantic change MUST update SSOT first and mirror into `nesp_erc.md`.
- Event names, state machine, invariants MUST remain consistent across both.
- Non-normative texts (whitepaper/game analysis) MUST NOT introduce new invariants.

Change Control
- Use pull requests with a clear rationale, diff to state machine, and impact on metrics (MET.*) and governance counters (GOV.*).
- Breaking changes should include a migration note in SSOT §Backwards Compatibility.

