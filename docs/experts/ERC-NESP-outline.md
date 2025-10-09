# ERC Proposal Outline — No-Arbitration Escrow Settlement Protocol (NESP)

## 1. Preamble
- EIP: *TBD*
- Title: ERC: No-Arbitration Escrow Settlement Protocol (NESP)
- Author(s): *Name + Contact*
- Status: Draft
- Type: Standards Track (ERC)
- Category: ERC
- Created: YYYY-MM-DD
- Requires: *e.g., ERC-2771, ERC-4337 (if applicable)*
- Replaces / Supersedes: *None*
- License: *e.g., CC0-1.0*

## 2. Abstract
- Summary of the protocol: chain-based escrow, no arbitration, timed dispute, symmetric forfeiture, zero protocol fee, third-party gifting.

## 3. Motivation
- Problems NESP solves; limitations of centralized escrow (Alipay) or arbitration DAO.
- Target scenarios: agent-to-agent trades, high-value tasks, marketplaces needing credible neutrality.
- Design goals: Minimal enshrinement（L1 只提供可验证状态/对称没收/Pull 结算）、symmetric deterrence、zero protocol fee（合约层不抽成，仅 Gas）、observable SLO。

## 4. Specification
### 4.1 Terminology & Roles
- Client (buyer), Contractor (seller), Resolved Subject, ForfeitPool, Escrow (E), Settlement Amount (A).
- States: Initialized, Executing, Reviewing, Disputing, Settled, Forfeited, Cancelled.

### 4.2 Overview
- Escrow flow, time anchors (`startTime`, `readyAt`, `disputeStart`) and windows (`D_due`, `D_rev`, `D_dis`).
- Pull settlement (balance credited then withdrawn), symmetric forfeiture when no agreement.

### 4.3 Function Interfaces
- `createOrder`, `createAndDeposit`, `depositEscrow`, `acceptOrder`, `markReady`, `approveReceipt`, `raiseDispute`, `settleWithSigs`, `timeoutSettle`, `timeoutForfeit`, `cancelOrder`, `withdraw`, `extendDue`, `extendReview`.
- Parameters, guards（Resolved Subject = 直连 `msg.sender` / 2771 `_msgSender()` / 4337 `userOp.sender`），每个函数引用主体约束表，列明返回值、副作用、触发事件。

### 4.4 Events
- `OrderCreated`, `EscrowDeposited`, `Accepted`, `DisputeRaised`, `Settled`, `AmountSettled`, `Forfeited`, `Cancelled`, `BalanceCredited`, `BalanceWithdrawn`.
- Field meanings（e.g., `actor`, `kind`）、sample topics，并注明这些字段构成“最小充分集合”，如需额外业务字段可通过自定义事件扩展。

### 4.5 State Machine & Guards
- Allowed transitions (E1–E13) mapping to functions, initiator, corresponding event.
- SIA (state invariant actions) such as `depositEscrow`, `extendDue`, `extendReview`.
- Resolved Subject definition及“主体约束表”（client/contractor/any）附带授权规则；第三方赠与路径需明确 `payer = subject`。

### 4.6 Invariants
- A ≤ E, zero protocol fee, pull settlement, timers monotonic, symmetric forfeiture, single-credit per order.

### 4.7 Optional Extensions / Hooks (Informative)
- Suggestion for optional metadata (artifact hash, etc.) while keeping core minimal.

## 5. Rationale
- Why symmetric forfeiture deters stall（拖延边际收益 = 0）、why minimal enshrinement（L1 仅托管+对称没收+事件）、why allow third-party gifting（自担风险且不改变权利义务）；comparison with arbitration DAO, HTLC, state channel。
- 关联 R1–R4 及反例映射：若破坏 A≤E → R1 失效；若允许延长 D_dis → R3 失效；若签名域缺项 → R4 失效。

## 6. Backwards Compatibility
- Interaction with existing escrow contracts, ERC-20 support, ETH flow.
- Migration considerations if upgrading from existing NESP deployments.

## 7. Security Considerations
- Signature domain (EIP-712/1271), replay prevention, multi-hop/ErrUnauthorized checks（建议错误签名/事件主题供审计使用）。
- Pull settlement + nonReentrant withdraw, CEI pattern（说明所有状态变更入口遵循 “checks-effects-interactions”）。
- Timer configuration 推奨区间（`D_dis >= 2 * T_reorg`；举例主网/侧链参考值），DOS/排序风险及 timeout* 护栏。
- Treat third-party deposit as gifting（payer = subject，需提前授予 allowance），无额外权利。

## 8. Operational Considerations / Monitoring (Informative)
- Metrics MET.1–MET.6, GOV.1–GOV.3 definitions, counting/de-dup rules（`EscrowDeposited`、`BalanceWithdrawn` 可重复，其余事件单次计）。
- SLO predicate `SLO_T(W)`, actions (stop-write/whitelist/rollback), CHG Runbooks.
- Observable anchors (events, logs) for audit; potential dashboards/panels。
- Residual risks & exit playbook（信息性）：触发条件、停写/白名单/回滚动作与 CHG Runbook 关联；说明隐私/日志披露最小化。

## 9. Reference Implementation
- Link or inline example contract (Solidity). Include signature verification, subject resolution helpers.

## 10. Test Cases
- Coverage for no-dispute success, signature settlement, timeout forfeiture, third-party gifting, 2771/4337 identity checks.
- Include fuzzy tests / integration tests.

## 11. Appendix
- State transition diagram, trace table (state → function → invariant → metric).
- Glossary of terms, scenario walkthrough (storyboard)。
- R1–R4 payoff reasoning & counterexamples；SLO/Runbook 示例；timeout 争议流程示意图（信息性）。

## 12. References
- EIPs and papers used (EIP-712, ERC-2771, ERC-4337, etc.), whitepaper sources, public discussions.

## 13. Copyright
- e.g., CC0-1.0 or other license per ERC process.
