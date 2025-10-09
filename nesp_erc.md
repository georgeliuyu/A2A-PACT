# ERC: No-Arbitration Escrow Settlement Protocol (NESP)

## 1. Preamble

- **EIP**: TBA  
- **Title**: ERC: No-Arbitration Escrow Settlement Protocol (NESP)  
- **Author(s)**: *TBA*  
- **Status**: Draft  
- **Type**: Standards Track (ERC)  
- **Category**: ERC  
- **Created**: 2025-10-09  
- **Requires**: ERC-20, ERC-712, ERC-2771, ERC-4337 (informative)  
- **Replaces**: None  
- **License**: CC0-1.0

## 2. Abstract

This proposal standardises a trust-minimised escrow settlement protocol for agent-to-agent (A2A) transactions. Buyers deposit funds into a smart-contract escrow before delivery. Sellers accept, fulfil, and can request settlement. If both parties agree, the escrow is released in full. If a dispute occurs, the parties negotiate an amount off-chain and co-sign a settlement message; otherwise, the escrow is forfeited symmetrically after a fixed dispute window. The protocol enforces zero protocol fees (gas only), supports third-party gifting/ top-ups, and emits auditable events for monitoring and service-level objectives (SLOs).

## 3. Motivation

Centralised escrow platforms (e.g., traditional marketplaces or payment intermediaries) introduce counterparty risk, custodial control, fees, and discretionary adjudication. Decentralised arbitration frameworks add governance complexity and require L1 social consensus. NESP provides a minimal on-chain core: escrow accounting, limited-time dispute resolution, and symmetric forfeiture. It is suited for:

- Once-off, subjective deliveries (digital goods, freelance tasks) where parties desire a neutral escrow without third-party arbitration.  
- Marketplaces and agent-based platforms that want programmable settlement yet preserve freedom for off-chain negotiation.  
- Scenarios demanding auditable, rule-based enforcement with zero protocol fees.

Design goals:

1. **Minimal Enshrinement** – L1 smart contract only records verified states, time anchors, escrow arithmetic, symmetric forfeiture, and emits audit events. No built-in arbitration or discretionary judgement.  
2. **Symmetric Deterrence** – if no agreement is reached before the dispute window closes, both parties lose the escrow; the marginal benefit of stalling is forced to zero.  
3. **Zero Protocol Fee** – the escrow contract must not take commission; all settlement paths satisfy `escrow_before = payout + refund` or `escrow_before = forfeited`. Only gas costs remain.  
4. **Observable SLO** – the contract emits a minimal but sufficient event set enabling monitoring of settlement latency, dispute frequency, gifting, and withdrawals.

## 4. Specification

### 4.1 Terminology & Roles

- **Client**: buyer funding the escrow.  
- **Contractor**: seller delivering the agreed work/goods.  
- **Resolved Subject**: the business-level caller resolved from a transaction (direct `msg.sender`, `ERC2771` `_msgSender()`, or `ERC4337` `userOp.sender`).  
- **Escrow (E)**: total funds deposited for the order, denominated in ETH or an ERC-20 token.  
- **Settlement Amount (A)**: amount released to contractor after settlement (`0 ≤ A ≤ E`).  
- **ForfeitPool**: logical bucket holding forfeited escrow (remains in contract balance; never redistributed).  
- **States**: `Initialized`, `Executing`, `Reviewing`, `Disputing`, `Settled`, `Forfeited`, `Cancelled`.  
- **Timers**: absolute anchors (`startTime`, `readyAt`, `disputeStart`) and configurable windows (`D_due`, `D_rev`, `D_dis`).

### 4.2 Data Structures

Implementations **MAY** use the following structure per `orderId`:

```solidity
struct Order {
    address client;
    address contractor;
    address token;            // address(0) for native ETH
    uint256 escrow;           // total deposited amount E
    uint256 dueSec;           // D_due in seconds
    uint256 revSec;           // D_rev in seconds
    uint256 disSec;           // D_dis in seconds
    uint64  startTime;
    uint64  readyAt;
    uint64  disputeStart;
    OrderState state;
}
```

Balance accounting for withdrawals should follow a pull model:

```solidity
mapping(address => mapping(address => uint256)) public withdrawable; // token => user => amount
```

### 4.3 Function Interfaces

All public functions **MUST** expose the following signatures. Implementations MAY extend them with modifiers or overloads as long as the minimal semantics are preserved.

| Function | Description | Allowed Subject |
|----------|-------------|-----------------|
| `createOrder(address token, address contractor, uint256 dueSec, uint256 revSec, uint256 disSec) returns (uint256 orderId)` | Create an order with negotiated windows. Allows zero inputs to adopt default template (e.g. 1d/1d/7d). | `subject == client` (creator) |
| `createAndDeposit(address token, address contractor, uint256 dueSec, uint256 revSec, uint256 disSec, uint256 amount)` payable | Create order and deposit `amount` in the same transaction. | `subject == client` |
| `depositEscrow(uint256 orderId, uint256 amount)` payable | Increase escrow. Any address may gift. If caller uses 2771/4337 path claiming to act for client, implementation MUST verify `subject == client`; otherwise treat as gifting (payer = subject). | `any` (with rules above) |
| `acceptOrder(uint256 orderId)` | Contractor accepts order, entering Executing. | `subject == contractor` |
| `markReady(uint256 orderId)` | Contractor declares delivery ready, sets `readyAt`. | `subject == contractor` |
| `approveReceipt(uint256 orderId)` | Client confirms receipt, triggers full settlement (`amountToSeller = E`). | `subject == client` |
| `raiseDispute(uint256 orderId)` | Either party moves order to Disputing, records `disputeStart`. | `subject ∈ {client, contractor}` |
| `settleWithSigs(uint256 orderId, bytes calldata payload, bytes calldata sigProposer, bytes calldata sigAcceptor)` | Settles dispute with co-signed amount payload (EIP-712 / 1271). | `subject ∈ {client, contractor}` (counterparty signature required) |
| `timeoutSettle(uint256 orderId)` | After review window, release full escrow to contractor. | `any` |
| `timeoutForfeit(uint256 orderId)` | After dispute window, forfeit escrow to ForfeitPool. | `any` |
| `cancelOrder(uint256 orderId)` | Cancel order. Depending on guard, only client/contractor can trigger. | `subject == client` (G.E6) or `subject == contractor` (G.E7/G.E11) |
| `withdraw(address token)` | Withdraw available balance for caller. | `any` |
| `extendDue(uint256 orderId, uint256 newDueSec)` | Client extends delivery window (strictly larger). | `subject == client` |
| `extendReview(uint256 orderId, uint256 newRevSec)` | Contractor extends review window (strictly larger). | `subject == contractor` |

**Escrow Behaviour**:
- Native ETH: require `msg.value == amount`.  
- ERC-20: require `msg.value == 0`; implementation MUST call `SafeERC20.safeTransferFrom(payer, address(this), amount)` where `payer = subject`. Gifting callers must have approved the contract beforehand.  
- Deposits are forbidden when `state ∈ {Disputing, Settled, Forfeited, Cancelled}`.

### 4.4 Events

| Event | Trigger | Purpose |
|-------|---------|---------|
| `OrderCreated(orderId, client, contractor, token, dueSec, revSec, disSec, ts)` | Order creation | Records participants, asset, effective windows |
| `EscrowDeposited(orderId, from, amount, newEscrow, ts, via)` | Successful deposit | Logs funding source and call path (`via = address(0)` for direct, forwarder, EntryPoint, etc.) |
| `Accepted(orderId, escrow, ts)` | Contractor accepts | Confirms order enters Executing and current escrow |
| `DisputeRaised(orderId, by, ts)` | Enter Disputing | Records initiator and timestamp |
| `Settled(orderId, amountToSeller, escrow, ts, actor)` | Full settlement (client confirmation or review timeout) | `actor ∈ {Client, Timeout}` distinguishes path |
| `AmountSettled(orderId, proposer, acceptor, amountToSeller, nonce, ts)` | Dispute resolved via mutual signatures | Contains proposer/acceptor, settlement amount, and nonce |
| `Forfeited(orderId, ts)` | Dispute window expired | Indicates symmetric forfeiture |
| `Cancelled(orderId, ts, cancelledBy)` | Order cancelled | `cancelledBy ∈ {Client, Contractor}` |
| `BalanceCredited(orderId, to, token, amount, kind, ts)` | Settlement/refund credited | `kind ∈ {Payout, Refund}` |
| `BalanceWithdrawn(to, token, amount, ts)` | Pull withdrawal success | Indicates actual transfer to user |

These events form the minimal sufficient set; implementers may emit additional events as long as they do not break invariants.

### 4.5 State Machine & Guards

**Resolved Subject**:  
- Direct call: `subject = msg.sender`, `via = address(0)`.  
- EIP-2771: if `isTrustedForwarder(msg.sender) == true`, `subject = _msgSender()`.  
- EIP-4337: if `msg.sender == EntryPoint`, `subject = userOp.sender`.

#### Allowed Transitions

| Transition | Function | Subject | Description |
|------------|----------|---------|-------------|
| E1 `Initialized → Executing` | `acceptOrder` | contractor | Set `startTime = block.timestamp`. |
| E2 `Initialized → Cancelled` | `cancelOrder` | client/contractor (with guards) | Order never accepted. |
| E3 `Executing → Reviewing` | `markReady` | contractor | Sets `readyAt = block.timestamp`. |
| E4 `Executing → Settled` | `approveReceipt` | client | Full payout (`A = E`). |
| E5 `Executing → Disputing` | `raiseDispute` | client/contractor | Sets `disputeStart`. |
| E6 `Executing → Cancelled` | `cancelOrder` | client (after `now ≥ startTime + D_due` and `readyAt` unset) | Refund. |
| E7 `Executing → Cancelled` | `cancelOrder` | contractor (unconditional) | Refund. |
| E8 `Reviewing → Settled` | `approveReceipt` | client | Full payout. |
| E9 `Reviewing → Settled` | `timeoutSettle` | any | When `now ≥ readyAt + D_rev`, full payout. |
| E10 `Reviewing → Disputing` | `raiseDispute` | client/contractor | Sets `disputeStart`. |
| E11 `Reviewing → Cancelled` | `cancelOrder` | contractor | Refund. |
| E12 `Disputing → Settled` | `settleWithSigs` | counterparty submits payload with both signatures | Partial payout (`0 ≤ A ≤ E`). |
| E13 `Disputing → Forfeited` | `timeoutForfeit` | any | When `now ≥ disputeStart + D_dis`, escrow forfeited.

#### State-Invariant Actions (SIA)

- SIA1 `extendDue(orderId, newDueSec)` requires `subject == client` and `newDueSec > current D_due`.  
- SIA2 `extendReview(orderId, newRevSec)` requires `subject == contractor` and `newRevSec > current D_rev`.  
- SIA3 `depositEscrow(orderId, amount)` requires `amount > 0`, `state ∈ {Initialized, Executing, Reviewing}`, and follows the gifting/allowance rules described in §4.3.

#### Timers & Anchors

- `startTime`, `readyAt`, `disputeStart` are monotonic: once set, MUST NOT change.  
- Windows `D_due` and `D_rev` may be extended (SIA1/2) but never decreased.  
- `D_dis` is immutable post creation and MUST satisfy `D_dis ≥ 2 * T_reorg` for the target chain (implementation MUST document recommended values).

### 4.6 Invariants

Implementations MUST enforce:

1. **Escrow monotonicity**: `escrow` (E) only increases via `depositEscrow`; no direct decrements.  
2. **Settlement bounds**: `0 ≤ amountToSeller ≤ escrow` for any settlement path.  
3. **Zero protocol fee**: for any settlement or forfeiture transition, `escrow_before = amountToSeller + refundToBuyer` or `escrow_before = forfeited`. Any deviation MUST revert with `ErrFeeForbidden`.  
4. **Single credit**: for each order, payouts/refunds are credited once (use a single-credit guard).  
5. **Pull settlement**: transfers to users happen only via `withdraw`; state transitions merely adjust internal balances.  
6. **Timeout precedence**: before executing external operations, implementations should check whether timeout conditions already apply (`timeoutSettle` / `timeoutForfeit`). If they do, those paths MUST be triggered or the call MUST revert, preventing strategic delay.  
7. **Forfeit pool isolation**: forfeited assets remain in the contract balance; no redistribution or burning.

### 4.7 Optional Extensions (Informative)

Implementations MAY provide:

- Metadata hooks (e.g., `markReady(orderId, bytes32 artifactHash)` emitting additional event).  
- Off-chain bundlers or indexers that enrich events with delivery proof, chat transcripts, or negotiation history.  
- Role-based access control for automated top-ups (e.g., DAO treasury).  
- Batch withdrawals or batched dispute settlements for gas efficiency.

## 5. Rationale

- **Symmetric Forfeiture**: by making “no agreement” outcome a mutual loss, neither party gains from stalling; marginal utility of delay is forced to zero, improving bargaining.  
- **Minimal Enshrinement**: L1 only stores escrow, timestamps, and emits events; arbitration, evidence, or credentialing remain off-chain. This aligns with credible neutrality.  
- **Third-Party Gifting**: allowing anyone to deposit (with gifting semantics) supports DAOs, friends-and-family sponsorships, or operational top-ups. Treating them as gifts avoids rewriting order ownership.  
- **Subject Guards**: mapping `subject` across direct calls, ERC-2771, and ERC-4337 ensures account abstraction wallets integrate without rewriting access control.  
- **Comparison**: centralised escrow (Alipay) relies on trust and fees; arbitration DAOs impose governance overhead; HTLCs only fit objective conditions; state channels require constant connectivity. NESP covers subjective, once-off transactions with minimal chain footprint.

## 6. Backwards Compatibility

Implementations compatible with previous NESP deployments can:

- Support default windows (1 day delivery/review, 7 day dispute) for contracts expecting prior behaviour.  
- Provide adapter contracts to read legacy order storage if needed.  
- Continue interacting with standard ERC-20 tokens; no changes to token contracts.  
- When upgrading, migrate outstanding balances and orders via one-time scripts, ensuring invariants hold post-migration.

## 7. Security Considerations

- **Signatures & Replay**: settlement payloads MUST use EIP-712 domain `{chainId, contract, orderId, tokenAddr, amountToSeller, proposer, acceptor, nonce, deadline}`. Nonces are per `(orderId, signer)` and MUST be single-use. Implementations should consider `IERC1271` for smart-contract wallets.  
- **Multi-hop / Unauthorized Calls**: deposit/accept functions MUST reject unauthorised multi-hop calls; recommended to expose `ErrUnauthorized()` error and rely on event absence (`EscrowDeposited`) as audit evidence.  
- **CEI & Re-entrancy**: state updates SHOULD follow checks-effects-interactions. `withdraw` MUST be `nonReentrant` and call tokens safely.  
- **Timer Configuration**: deployments MUST choose `D_due`, `D_rev`, `D_dis` with `D_dis >= 2 * T_reorg` (e.g., >= 2 minutes on mainnet). Shorter windows risk reorg exploitation.  
- **Gift Deposits**: third-party deposits are unconditional gifts; they confer no rights. Callers must approve tokens before gifting.  
- **DOS / Gas Considerations**: storing large numbers of orders is O(1) per order. Implementers should guard against unbounded loops (e.g., batch operations).  
- **Dispute Signatures**: ensure all payloads include expiry (`deadline`) to avoid indefinite sign reuse.

## 8. Operational Considerations (Informative)

- **Metrics**: recommended to track MET.1 settlement latency P95, MET.2 withdrawal failure rate, MET.3 locked balances, MET.4 acceptance rate, MET.5 fee violations (should be zero), MET.6 state transition latency; GOV.1 success vs forfeited vs cancelled, GOV.2 ratio `A/E`, GOV.3 dispute duration distribution.  
- **Counting Rules**: `EscrowDeposited` and `BalanceWithdrawn` can repeat per order (gifting, multiple withdrawals); other events count once.  
- **SLO Predicate**: `SLO_T(W) := (MET.5 = 0) ∧ (forfeit_rate ≤ θ) ∧ (acceptance_rate ≥ β) ∧ (p95_settle ≤ τ)`. Exceeding thresholds triggers CHG:SLO-Runbook actions (stop-write, whitelist, rollback).  
- **Residual Risk & Exit Playbook**: Entities should document triggers (e.g., spike in forfeiture), actions (pause intake, enable whitelisting, roll back release), and contacts, keeping logs minimal.  
- **Dashboards**: indexer teams can build panels from emissions to monitor flows, top-ups, disputes, and questionably high forfeiture rates.

## 9. Reference Implementation

A minimal Solidity reference implementation (abridged):

```solidity
contract NESP {
    enum State { Initialized, Executing, Reviewing, Disputing, Settled, Forfeited, Cancelled }
    mapping(address => bool) public trustedForwarders; // configure trusted forwarders (ERC-2771)
    address public entryPoint; // optional ERC-4337 EntryPoint address
    struct Order { address client; address contractor; address token; uint256 escrow;
        uint256 dueSec; uint256 revSec; uint256 disSec; uint64 startTime; uint64 readyAt; uint64 disputeStart; State state; }
    mapping(uint256 => Order) public orders; uint256 public nextOrderId;
    mapping(address => mapping(address => uint256)) public withdrawable;

    function _subject() internal view returns (address subject, address via) {
        // implementers fill with ERC2771/4337 detection
    }

    function _isTrustedPath(address via) internal view returns (bool) {
        // Example implementation: only treat known forwarders / EntryPoint as trusted. Direct calls (via == address(0)) are not subject to this check.
        if (via == address(0)) return false;
        return trustedForwarders[via] || via == address(entryPoint);
    }

    function depositEscrow(uint256 orderId, uint256 amount) external payable {
        (address subject, address via) = _subject();
        Order storage order = orders[orderId];
        require(order.state != State.Disputing && order.state < State.Settled, "frozen");
        bool trustedPath = _isTrustedPath(via);
        if (trustedPath && subject != order.client) revert("not client");
        if (order.token == address(0)) {
            require(msg.value == amount, "bad value");
        } else {
            require(msg.value == 0, "bad ether");
            IERC20(order.token).transferFrom(subject, address(this), amount);
        }
        order.escrow += amount;
        emit EscrowDeposited(orderId, subject, amount, order.escrow, block.timestamp, via);
    }

    // ... remaining functions enforcing guards and invariants
}
```

A full implementation SHOULD include subject resolution helpers, signature verification, timeout checks, and withdrawal logic. Testing MUST cover all invariants.

## 10. Test Cases

Implementers SHOULD provide tests covering:

1. **Straight-through settlement**: create → deposit → accept → approveReceipt → withdraw.  
2. **Signature settlement**: dispute + co-signed payload with valid EIP-712 typed data.  
3. **Symmetric forfeiture**: dispute without agreement until `disputeStart + D_dis`.  
4. **Third-party gifting**: non-client deposits ERC-20 and ETH, verifying withdrawal semantics.  
5. **Subject guards**: ERC-2771 forwarder and ERC-4337 EntryPoint flows, ensuring only authorised subjects pass.  
6. **Timeout precedence**: ensure re-entrancy or late calls can't skip `timeout*`.  
7. **Zero-fee invariant**: confirm any violation reverts with `ErrFeeForbidden`.  
8. **Withdrawal idempotence**: repeated withdraw after balance zero has no effect.

## 11. Appendix

- **State Diagram**: depict transitions E1–E13 with functions and guards.  
- **Trace Table**: `state → function → invariant → metric/event`.  
- **R1–R4 Proof Sketch**: show why rational actors prefer timely cooperation; include counterexamples (Ex-1 – Ex-3).  
- **SLO & Runbook Templates**: example CHG entries for thresholds, actions, rollback steps.  
- **Timeout Flow Example**: timeline illustrating dispute, extension, settlement, forfeiture.

## 12. References

1. Vitalik Buterin, “Endgame” & minimal enshrinement posts.  
2. ERC-2771: Secure Protocol for Native Meta-Transactions.  
3. ERC-4337: Account Abstraction Using Alt Mempool.  
4. ERC-20, ERC-712 standards.  
5. NESP Whitepaper (2025-10-09).  
6. Public discussions on symmetric forfeiture escrow (links to community threads).

## 13. Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

