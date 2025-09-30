# PACT 共识（阶段性）

版本：0.1（草案）  日期：2025-09-30
参与：Vitalik、Tim、Nenad、Glen、George、白隼（评审）

---

## 核心共识（Core）
- 最小内置：链上仅做“资金 + 状态 + 时钟”；不内置仲裁、信誉与治理投票。
- 一次性结清：每个订单恰一次成功提现；不支持分期多次支付。
- 统一金额口径：A ≤ E_c(t_settle)；托管 E_c 单调不减；Top‑up 仅允许增加；终态不可再变更。
- Pull 出金：状态变更仅记账，实际转账通过独立提现完成，保证幂等与审计。
- 公共超时与对称威慑：评审/争议有硬截止；没收可达但非优势策略，形成威慑以促成合作。
- 可验证协商：争议阶段仅就“金额”协商，采用 EIP‑712/1271 离线报价 + 对手方链上接受。
- 可追溯与可重放：事件/状态可索引与回放；每条状态转换可映射到接口/事件/不变量。
- L2 友好：优先在 Rollup/L2 推广；L1 仅做安全与数据可用层。

## 标准化共识（ERC）
- 分层发现：ERC‑165 接口分层。
  - Core（MUST）：创建、接单、推进（ready/approve/dispute/timeout/forfeit/cancel）、单次提现。
  - Query（MUST）：状态与时钟查询、可触发性提示。
  - SettlementPrice（MAY）：EIP‑712 价格型接受（Disputing 专用）。
  - TopUp（MAY）：client‑only 加押（非终态）。
  - ReviewExtension（MAY）：contractor 延长评审期（仅 Reviewing）。
- 非目标（Out of Scope）：分期结清、链上仲裁/信誉/治理聚合、复杂代币语义（fee‑on‑transfer/rebasing/777 hooks）。

## 安全共识（Security）
- 重入与 CEI：使用 ReentrancyGuard + Checks‑Effects‑Interactions。
- 签名安全：域分隔、nonce、deadline、防重放；EIP‑1271 支持合约账户；验对手方。
- 时间竞态：入口前抢占到期（超时先行处理）；时间以 block.timestamp 计。
- 代币安全：默认仅支持标准 ERC‑20 与 ETH；余额差核验；不支持异常语义代币。
- 幂等：提现单次幂等；所有关键状态跃迁具幂等标记与回滚路径。

## 指标与评测共识（Observability & Evaluation）
- KPI：结清率、争议率与时长（中位/99p）、A/E_c 分布、Top‑up 频率、自动结清/没收占比、失败/回滚率、Gas 分布。
- 基线对照：直接支付、平台托管（中心化仲裁/分期）、PACT（一次性结清 + 可协商）。
- 对手模型：理性、半理性、对抗（重放/抢跑/延时/余额噪声）。
- 可复现：Foundry/Hardhat 单元 + 属性测试；事件回放脚本与仿真器。

## 文档一致性（SSOT）
- 规范唯一语义源：`pact_spec.md`。
- 白皮书：`whitepaper/PACT_Whitepaper.md`（架构与生态说明）。
- 专家观点：`whitepaper/expert_views/`。

---

（说明）本文件记录阶段性“共识”，用于指导后续白皮书与 ERC 草案推进；争议与未决事项在 debate/ 记录。

