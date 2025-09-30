# PACT 白皮书

去中心化结算基座：最小托管与争议结算（“去中心化支付宝”）

版本：0.1（草案）
发布日期：2025-09-30
作者：A2A‑PACT 工作组（征集中）

---

## 摘要（Abstract）

本文提出 PACT，一种面向代理—代理（Agent‑to‑Agent, A2A）虚拟经济的“最小托管与争议结算”协议，将链上职责收敛为资金、状态与时钟三要素。PACT 以统一金额口径 A ≤ E_c(t)、一次性结清（非分期）、公共超时与对称没收构建可信中立的结算基座；在争议阶段，采用 EIP‑712/1271 的离线报价接受实现可验证协商；在出金侧，采用 Pull 结算确保可审计与幂等。与把仲裁/信誉/治理上链的方案不同，PACT 遵循“最小内置（Minimal Enshrinement）”，把复杂社会层逻辑留给上层生态与链下治理。

在 Virtual Agent Economies（VAE，Google/DeepMind）“起源×渗透性”框架下，我们论证 PACT 能提供“可渗透但可控”的市场基础设施：通过单调加押、硬截止、公共事件与可重放状态，支撑代理大规模、跨域、频繁的结算需求，同时保留审计与熔断的操作空间。结合形式化状态机与不变量，我们给出安全与经济性分析，并展示原型与仿真下的初步证据：在理性或半理性对手模型中，PACT 提高结清率、缩短争议时长，并将极端没收约束为威慑而非常态。我们建议将该协议标准化为 ERC‑PACT，供钱包、Rollup、代理框架与市场统一复用。

---

## 1. 背景与动机（Motivation）

- 代理虚拟经济（VAE）趋势：多主体在超人类频率下协作与交易，需要高吞吐、可组合、可审计的结算底座。
- 现有缺口：
  - 中心化托管（平台型“支付宝”）牺牲可组合性与可验证性；
  - 把仲裁/治理内置到共识层易过载共识，放大系统性风险；
  - 分期/多次支付增加状态爆炸与攻击面，不利于审计与幂等。
- 目标：提供一个“去中心化支付宝”式的最小协议，具备跨场景可复用的托管、争议与一次性结清能力，同时为上层留足治理与信誉空间。

贡献概述：
- 定义统一金额口径与状态机，给出形式化不变量与守卫；
- 提出 EIP‑712/1271 的可验证协商路径与 Pull 出金结算；
- 规范接口与事件（ERC‑PACT 草案），给出参考实现与一致性测试计划；
- 将协议映射到 VAE 的治理旋钮（渗透性/熔断/审计回放）。

---

## 2. 设计原则（Design Principles）

- 最小内置（MUST）：链上仅负责“资金+状态+时钟”，不内置仲裁/信誉/治理投票。
- 单一金额口径（MUST）：A ≤ E_c(t)，无独立 P；Top‑up 单调，提现单次。
- 一次性结清（MUST）：每个订单恰一次成功提现，不支持分期多次支付。
- 公共超时与对称威慑（MUST）：评审/争议时钟与超时动作公开、可触发；没收可达但非优势策略。
- 可验证协商（SHOULD）：争议阶段以 EIP‑712/1271 离线报价 + 对手方接受一次性结清。
- Pull 出金（MUST）：状态机阶段仅记账“可领额”，实际转账在独立提现中完成。
- 可重放与可追溯（MUST）：事件与状态可重放/索引；每条状态转换可追溯到接口与不变量。
- 生态兼容（SHOULD）：ERC‑165 发现、ERC‑20/ETH、4337 钱包友好、L2/索引器可用。

---

## 3. 模型与记号（Model and Notation）

- 参与者：Client（买方）、Contractor（卖方）。
- 金额：E_c(t) 托管额（单调不减）、A 实际结清额（0 ≤ A ≤ E_c(t_settle)）。
- 时间：相对时长 T_due/T_rev/T_dis；锚点 startTime/readyAt/disputeStart（一次性设置，禁止回拨）。
- 状态：Initialized, Executing, Reviewing, Disputing, Settled, Forfeited, Cancelled。
- SIA（状态不变动作）：extendDue（Client）、extendReview（Contractor）、topUpEscrow（Client）。
- 口径与单位：token 最小单位，时间以 block.timestamp 计。

不变量（部分）：
- INV.1 A ≤ E_c(t_settle)；
- INV.2 E_c 单调不减；
- INV.3 单次提现（payout/refund 幂等）；
- INV.4 T_due/T_rev/T_dis 仅允许延长；
- INV.5 锚点一旦设置禁止回拨；
- INV.6 终态不可再 Top‑up 或状态转换。

---

## 4. 状态机与守卫（State Machine, Guards）

允许的转换（节选，完整见附录 A 与 pact_spec.md）：
- E1 Initialized —acceptOrder→ Executing（contractor；副作用：startTime=now）
- E3 Executing —markReady→ Reviewing（contractor；guard：now < startTime+T_due；副作用：readyAt=now，重置 T_rev）
- E4/E9 Executing/Reviewing —approveReceipt→ Settled（client）
- E10 Reviewing —timeoutSettle→ Settled（any；guard：now ≥ readyAt+T_rev）
- E5/E11 任一 —raiseDispute→ Disputing（either；副作用：disputeStart=now）
- E14 Disputing —acceptPriceBySig→ Settled（对手方；验 712/1271、nonce、deadline）
- E15 Disputing —timeoutForfeit→ Forfeited（any；guard：now ≥ disputeStart+T_dis）
- E8/E13 任一 —requestForfeit→ Forfeited（either）
- 取消：Initialized 双向；Executing 在“从未 Ready 且到期”由 client 允许；contractor 可随时取消。

SIA（状态不变动作）：
- extendDue（client；严格延后）；extendReview（contractor；严格延后）；topUpEscrow（client；正金额；非终态）。

---

## 5. 机制语义与实现要点

- 入金与托管：初始化入金或后续 Top‑up，E_c 单调增加；禁止减少。
- 一次性结清：
  - 审核通过/超时结清：amountToSeller = escrow；refundToBuyer = escrow - amountToSeller；
  - 争议协商：acceptPriceBySig(amountToSeller)；refund 为差额；
  - 结清时刻 t_settle 以 approveReceipt/timeoutSettle/acceptPriceBySig 的链上确认时间定义。
- Pull 出金：payout/refund 由各自地址单独提现；提现幂等且最多一次成功。
- 超时前抢占：外部入口前先行处理到期的自动结清/没收，防“过期后再调用”的竞态。
- 事件与可追溯：每次状态变更与 SIA/提现均发事件；事件设计满足索引与回放。

---

## 6. 经济性与博弈（Game‑Theoretic Intuition）

- 买方“付款 vs 没收”：u_C(pay) − u_C(forfeit) = E_c − A ≥ 0（当 A < E_c 时严格 >0）。
- 卖方“继续/提交 vs 立即没收”：若剩余成本 R(t) < A，则继续/提交严格优于没收；
- 沉默代价：评审超时自动结清，争议超时自动没收；时间成本 d_i(t) 使“拖延”自损；
- 结论：极端动作可达但非优势；理性路径是“及时 Ready → 付款或评审/协商 → 结清”。

---

## 7. 接口与事件（ERC‑PACT 概览）

- 分层接口（ERC‑165 发现）：
  - Core（MUST）：创建、接单、推进（ready/approve/dispute/timeout/forfeit/cancel）、单次提现；
  - Query（MUST）：状态/时钟/可触发性查询；
  - SettlementPrice（MAY）：争议阶段的 EIP‑712 报价接受；
  - TopUp（MAY）：client‑only 加押；
  - ReviewExtension（MAY）：contractor 延长评审期。
- 事件（示例）：OrderCreated/Started/Ready/ReceiptApproved/DisputeOpened/SettlementAccepted/AutoSettled/Forfeited/Cancelled/PayoutWithdrawn/RefundWithdrawn/ReviewExtended/EscrowToppedUp/ForfeitRequested。
- 错误码（示例）：ERR.State/ERR.Timeout/ERR.Amount/ERR.Signature/ERR.Reentrancy 等。

参考接口草案见仓库 `pact_erc.md`；标准化文本将迁移至 `eips/erc-pact.md`。

---

## 8. 签名与消息格式（EIP‑712/1271）

- 域分隔（Domain Separator）：name=ERC‑PACT、version、chainId、verifyingContract、salt（可选）。
- PriceOffer 结构：{ orderId, amountToSeller, nonce, deadline }；
- 接受者必须为对手方；校验 nonce 单调与 deadline；
- 兼容 EIP‑1271：支持合约账户签名验证；
- 示例与测试向量将收录于 `specs/pact_eip712.md`（后续提交）。

---

## 9. 安全与合规（Security & Safety）

- ReentrancyGuard + CEI；
- 余额差核验与 SafeERC20；
- 单次提现不变量与幂等标记；
- 超时前抢占；
- 签名防重放：per‑order per‑proposer nonce，deadline，domain 绑定；
- 时间来源：block.timestamp 抖动不作为争议判断依据；
- 资产兼容：默认不支持 fee‑on‑transfer/rebasing/777 hooks；
- 审计与可观测：事件齐全、状态可重放、链下监控指标（见 §10）。

---

## 10. 指标与 SLO（Observability & KPIs）

- 结清率、争议率与中位/99p 时长；
- A/E_c 比率与分布、Top‑up 频率与时点；
- 极端动作（没收/取消）发生率；
- 自动结清/没收触发占比；
- Gas 成本分布与关键路径上限；
- 失败/回滚率与原因分类（签名/时序/余额不足/重入拦截等）。

---

## 11. 评测方法（Evaluation）

- 基线：
  - 直接支付（无托管）；
  - 平台内置托管（中心化仲裁/分期）；
  - PACT（一次性结清 + 可验证协商）。
- 对手模型：理性/半理性/对抗（重放/抢跑/延时）；
- 自变量：T_due/T_rev/T_dis、报价窗口、Top‑up 策略；
- 指标：见 §10；
- 工具链：Foundry/Hardhat 单元 + 属性测试、事件回放脚本、仿真器（可从 CSV/合成流量驱动）。

---

## 12. 生态与治理（Ecosystem & Governance）

- 钱包与账户抽象：4337 兼容（UserOp 发起/社恢/限权），支持 1271 验证；
- Rollup 与索引：事件可重放、状态可索引；
- 渗透性旋钮（VAE）：桥接额度、清算周期（T+N）、专用记账单位（可选）；
- 审计回放与熔断：争议激增/风控触发时，允许暂停新单、延长 T_rev/T_dis、限制 Top‑up；
- 合规考量：资金托管与提现的账务可证明、可追溯，避免协议承担实质仲裁职责。

---

## 13. 参考实现与部署（Reference Implementation）

- 代码结构建议：
  - contracts/: Core/Query/扩展接口与实现；
  - tests/: Conformance 与 Fuzz；
  - docs/: threat_model.md、trace_map.md、gas_slo.md；
  - eips/: erc-pact.md（草案）。
- 部署与集成：先在 L2 公测网灰度，开放事件订阅与仪表盘；达标后主网试点。

---

## 14. 相关工作（Related Work）

- A2A 协议与代理市场；
- 托管/仲裁合约（多签仲裁、Kleros 等）与其对比；
- 费用与钱包生态（EIP‑1559、ERC‑4337）；
- DeepMind: Virtual Agent Economies（VAE）框架与两维治理抓手。

---

## 15. 路线图与标准化（Roadmap）

- R1 规范冻结：pact_spec.md 正式版（已完成）；
- R2 EIP 草案：迁移并补齐接口 ID/事件/兼容性/参考实现链接；
- R3 参考实现 + 一致性测试：Core/Query/SettlementPrice 与 EIP‑712 向量；
- R4 仿真与指标：复现实验与对照；
- R5 白皮书定稿与社区评审；
- R6 提交 EIPs PR 与生态对接（钱包/市场/代理框架）。

---

## 16. 术语表（Glossary）

- A：实际结清额；E_c(t)：托管额；
- T_due/T_rev/T_dis：到期/评审/争议相对时长；
- Pull 结算：提现时转账；
- EIP‑712/1271：Typed‑Data 签名与合约账户签名验证。

---

## 附录 A：状态转换与追溯（Traceability）

- 追溯原则：每条 E.x 映射到至少一个 API/事件，覆盖其适用不变量；
- 例：E10 timeoutSettle → API.triggerAutoReceiptConfirm → EVT.AutoSettled；覆盖 INV.1/INV.3/INV.4。
- 完整映射表见 `docs/trace_map.md`（后续提交）与 `pact_spec.md` 第 3 章。

---

## 附录 B：签名与消息（示例）

- Domain：name=ERC‑PACT, version=1, chainId, verifyingContract；
- PriceOffer：{orderId, amountToSeller, nonce, deadline}；
- digest = keccak256(\x19\x01, domainSeparator, hashStruct(PriceOffer, ...))；
- 验证顺序：域→结构→签名→deadline→nonce→对手方。

---

## 附录 C：安全清单（最小）

- Reentrancy（重入）
- Signature Replay/Malleability（重放/可塑）
- Timestamp/Deadline Races（时间竞态）
- Token Semantics（代币语义偏差）
- Single‑Use Payout Idempotency（单次提现幂等）
- Pre‑Emptive Timeout Handling（入口前抢占到期）

---

## 附录 D：专家视角要点（Expert Views）

- Vitalik Buterin：whitepaper/expert_views/vitalik_buterin.md
- Tim Roughgarden：whitepaper/expert_views/tim_roughgarden.md
- Nenad Tomasev：whitepaper/expert_views/nenad_tomasev.md
- E. Glen Weyl：whitepaper/expert_views/glen_weyl.md
- 白隼（Blade Reviewer）：whitepaper/expert_views/blade_reviewer.md

（注）各视角从“最小内置/机制稳健/VAE 治理/多元协作/评审可执行”出发，为 PACT 白皮书提供关键要点。

注：本白皮书与 `pact_spec.md` 一致，作为架构与生态说明文档；规范性细节以 `pact_spec.md` 为唯一语义源（SSOT）。
