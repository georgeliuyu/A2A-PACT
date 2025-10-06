# NAS 协议（A2A‑NAS）白皮书 — Vitalik 版本

副题：Trust‑Minimized · Symmetric‑Forfeit（conditional）· Timed‑Dispute

别名：曾用名 PACT（Preemptive Accountability via Escrow）
Timezone=UTC

---

## 摘要（Abstract）

在不把“社会仲裁/声誉”塞进 L1 共识的前提下，NAS 协议（A2A‑NAS，No‑Arbitration Settlement）为 A2A 场景提供最小可行的问责与结算底座：以“签名定价（EIP‑712/1271）+ 限时争议（T_dis）+ 拉取式结算（Pull）”替代仲裁路径，并在对称条件下（conditional）用“对称没收（Symmetric‑Forfeit）”抑制廉价整蛊。协议以 nas_spec 为 SSOT，证明与测量以 nas_game_theory 为语义来源：存在子博弈完美均衡（SPNE）、有限步终止（活性）、阈值结构（A≤V+E_c）、在对称条件下的整蛊上界（G≤1）、以及通过 T1–T5 变换获得的约束最优性。白皮书坚持 Minimal Enshrinement 与 Credible Neutrality：仅提供可验证状态与记账；唯一通过判据为 MET.4（G 的 P50≤1.2），其余观测/对照/决策工具均为信息性，不纳入通过判据集。

---

## 1. 引言（问题与约束）

### 1.1 背景与动机

在 A2A 生态中，参与方缺乏可用的社会裁决与身份基础设施，传统“平台仲裁/声誉体系”不适用或成本高企。NAS 以可验证的合约状态与最小规则替代仲裁：将“事后裁决”变为“事前问责与可结算”。

### 1.2 最小不可违约束（A1–A8，来自 nas_spec）

- A1 无仲裁（No‑Arbitration）：不内置第三方对主观质量的裁决路径。
- A2 仅金额结算：结果只体现为链上金额移动；无强制履约/声誉惩罚。
- A3 低成本身份：默认不依赖 KYC/SBT；如需使用，属于外部叠加层。
- A4 公共时钟：以 block.timestamp 为共同时间基准。
- A5 有界排序：部署方给出 T_reorg 估计，要求 T_dis ≥ 2·T_reorg。
- A6 拉取式结算（Pull）：状态变更仅记账，实际转账在提现时发生。
- A7 可验证动作：报价签名使用 EIP‑712/1271。
- A8 争议期不可延长：T_dis 固定；T_due/T_rev 仅可“向后”延长且在争议前生效。

---

## 2. 协议概览（来自 nas_spec）

### 2.1 状态机与锚点

- 状态与转换：E1–E16（Initialized/Executing/Reviewing/Disputing/Settled/Forfeited/Cancelled）。
- 锚点与时长：startTime/readyAt/disputeStart；T_due/T_rev/T_dis（单位：秒）。
- 不变量：A ≤ E_c；单次提现；Forfeited 清零；Pull 语义等（INV.*）。

### 2.2 三要素

- 签名定价（EIP‑712/1271）：任何一方可提出签名要约，对手接受后按签名定价结算（E14）。
- 限时争议（T_dis）：Disputing 态至多持续 T_dis；超时则进入 Forfeited（E15）。
- 对称没收（Symmetric‑Forfeit，conditional）：在对称托管/机会成本近似对称时，整蛊上界 G≤1（见 §3.4）。

### 2.3 事件与日志（最小集）

事件字段最小集（信息性推荐）：`orderId, proposer, acceptor, amountToSeller, ts`；以最小披露达成可追溯与审计。

---

## 3. 核心定理（来自 nas_game_theory）

### 3.1 定理 1：SPNE 存在

有限期+完美信息+确定性支付/没收 → 回溯归纳（Kuhn）得到子博弈完美均衡（SPNE）。

### 3.2 定理 2：活性（有限步终止）

任一路径在有限真实时间内终止，上界 T_due+T_rev+T_dis；以势函数（剩余时间）证明单调递减至终态。

### 3.3 定理 3：阈值均衡

争议态买方接受价的阈值满足 A ≤ V+E_c；比较静态：∂/∂V>0，∂/∂E_c>0；关于时间项的符号以信息性回归观察（MET.5），不作通过判据。

### 3.4 定理 4：对称整蛊边界（conditional）

在对称托管与机会成本近似对称时，整蛊因子 G ≤ 1；若条件不满足，G 可能 >1，但存在无量纲上界。

### 3.5 定理 5：变换式约束最优性

通过 T1–T5（去分期/对称化/公共超时/仅金额/预先协商）将任意满足 A1–A8 的机制变换至 NAS，逐步不劣，传递性得到约束最优性。

---

## 4. 安全与可信中立（Credible Neutrality）

### 4.1 Minimal Enshrinement

NAS 不把社会仲裁/声誉/治理投票塞进共识层；共识层仅承载最小可验证状态与记账（Pull）。

### 4.2 时间与排序的鲁棒性

部署方应按目标链给出 T_reorg 估计，并设定 T_dis ≥ 2·T_reorg。对 L2/rollup，仅给出信息性适配说明，规范保持与 nas_spec 一致。

### 4.3 对称没收的条件性声明

“Symmetric‑Forfeit”严格为 conditional：对称托管/机会成本近似对称时成立；否则作为信息性诊断与治理输入（见 §8）。

---

## 5. 对照与决策支持（信息性）

- 基线 1：直接支付（无机制）— 事后效率不可实现；社会福利塌缩。
- 基线 2：中心化托管+仲裁（类外对照）— 成本构成与路径依赖，非判据。
- 基线 3：HTLC/状态通道— 适用边界明确；不适于陌生方的主观性交付。
- 决策树（信息性）：帮助在 NAS/HTLC/中心化之间抉择；不纳入 §6 判据集。

---

## 6. 观测与判据（唯一通过判据）

### 6.1 覆盖范围声明

本章判据仅覆盖 NAS 交易事件集合；比较表/决策树等为信息性，**不纳入**通过判据集。

### 6.2 判据与目标

- MET.4：整蛊观测 G 的目标统计量为 P50 ≤ 1.2（唯一通过判据）。
- MET.5：阈值回归信息性记录（price ~ E_c + T_dis，记录 R²与符号方向），**不纳入**通过判据。

### 6.3 复现与抽样（统一口径）

1) 观察窗口：最近 24h；如 n<200 则扩至 7d 或直至 n≥200。  
2) 抽样方法：连续时间窗口内全量；如总量过大，系统抽样步长 k=⌈N/200⌉（N=窗口内 NAS 订单总数）。  
3) 单位统一：时间字段（含 T_dis）统一“秒”；统计字段以事件日志为准。  
4) 工件标识：合约地址、观察窗口起止区块、参数变更交易哈希（如有）、告警触发/通知时间戳与通知渠道标识（优先级：消息哈希 > 渠道ID > 渠道名；介质=邮件/IM/工单之一）。

---

## 7. 实现与互操作（信息性）

### 7.1 签名定价（EIP‑712/1271）域与撤销语义

- 域字段：`chainId, orderId, amountToSeller, proposer, nonce, deadline`（建议最小集）。
- 覆盖/撤销：不提供“链上撤销入口”；以“新要约 + nonce 递增 + deadline 过期”实现覆盖与过期控制。
- 事件字段：`orderId, proposer, acceptor, amountToSeller, ts`（建议最小集）。

### 7.2 与 ERC‑4337/AA 的关系

- 代付/中继：不改变 Pull 结算语义；代付仅影响谁支付 gas 与谁发起交易；信任边界明确为中继/代付层（信息性）。

### 7.3 L2/跨域适配

- 由部署方给出 T_reorg 估计与执行域配置；白皮书仅提供信息性说明。

---

## 8. 公平与治理（信息性）

### 8.1 对称没收（conditional）与治理响应

- 条件性：对称托管/机会成本近似对称时，G≤1 成立；否则可能 >1。  
- 治理响应：当长窗 P50 偏离占比过高（信息性阈值）时，进入治理评审（例如上调 E_c、引入卖方风险约束、冷却限流等），**不改变**规范判据。

### 8.2 分群体诊断（信息性）

观察不同群体在 G 分布/争议尾部的差异，仅作为信息性输入；避免将“群体切片”升格为规范性通过判据。

---

## 9. 运维与版本（信息性）

### 9.1 监测与 SLO（示例）

- P99 时长（≤ T_due+T_rev+T_dis）、结清/争议/没收率、资金滞留余额等。

### 9.2 三层告警与回滚（单页流程）

观测 → 阈值 → 动作（通知/参数调整/回滚/冷却/限流），与 MET.4 判据分离。

### 9.3 版本化与变更卡

SSOT=nas_spec；任何状态机/INV/API/MET 变更按语义版本管理；变更卡说明影响面/迁移与兼容窗口。

---

## 10. 路线图（信息性）

- MVP：状态机完整、Pull 结算、事件最小集与不变量。
- 强化：批量提现/自动领取/代付；资产适配层与白名单。
- 互操作与标准化：接口标准与生态协作；保持 Minimal Enshrinement。

---

## 附录 A：符号与术语

V/C/E_c/A/T_*、事件与状态、MET/GOV 指标、命名与单位约定（时间统一“秒”）。

---

## 附录 B：Trace 映射（示例）

E14（Disputing→Settled，acceptPriceBySig）→ API: `acceptPriceBySig` → INV.2 → EVT: `PriceSettled/Settled` → MET: MET.4（报价接受率）。

---

## 附录 C：接口清单（信息性）

- 签名域：`chainId, orderId, amountToSeller, proposer, nonce, deadline`；nonce 作用域至少为 `{orderId, proposer}`；deadline 防过期与重放。
- 事件字段：`orderId, proposer, acceptor, amountToSeller, ts`；最小披露、可审计。
- 安全基线：nonReentrant + CEI；入口前抢占 timeout*；失败重试与去重（≤3 行流程）。

---

## 附录 D：观测与复现（信息性）

1) 窗口：24h（n<200 → 7d 或 n≥200）。  
2) 抽样：系统抽样 k=⌈N/200⌉（N=窗口内 NAS 订单总数）。  
3) 单位：时间字段统一“秒”；按事件日志统计。  
4) 工件：地址/起止区块/参数变更交易哈希/通知哈希优先。  
5) 判据：MET.4 P50≤1.2；MET.5 仅记录 R²与符号方向（不作为通过判据）。

---

参考与语义来源：

- 规范（SSOT）：docs/zh/nas_spec.md  
- 博弈与测量：docs/zh/nas_game_theory.md

