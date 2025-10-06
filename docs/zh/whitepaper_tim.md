# NAS 协议（A2A‑NAS）白皮书 — Tim 版本

副题：Trust‑Minimized · Symmetric‑Forfeit（conditional）· Timed‑Dispute

别名：曾用名 PACT（Preemptive Accountability via Escrow）
Timezone=UTC

---

## 摘要（Abstract）

从算法博弈论与机制设计角度，NAS 协议（A2A‑NAS，No‑Arbitration Settlement）在信任最小化约束下，以“签名定价 + 限时争议 + 拉取式结算”实现 A2A 交易的可问责与可结算。在对称条件下（conditional），对称没收将整蛊上界收敛为 G≤1。本文以 nas_spec 为规范性唯一语义源（SSOT），以 nas_game_theory 为博弈与测量的语义来源，陈述五个核心定理：SPNE 存在、有限步终止（活性）、阈值结构（A≤V+E_c）、对称整蛊上界（conditional）与 T1–T5 变换式约束最优性。唯一通过判据为 MET.4（G 的 P50≤1.2）；MET.5（阈值回归）为信息性记录；判据覆盖范围仅限 NAS 交易事件集合。

---

## 1. 形式化模型与约束（来自 nas_spec / nas_game_theory）

### 1.1 模型与记号

- 博弈 G=(N,S,A,U,I)，完美信息、有限期；参与者 N={Client, Contractor}。  
- 资金与结算：托管 E_c，结清金额 A，满足 A≤E_c；收益（V−A, A−C）。  
- 状态机 E1–E16，锚点 startTime/readyAt/disputeStart，时长 T_due/T_rev/T_dis。  
- 不变量（示例）：INV.1/2（结清口径）、INV.3（退款）、INV.8（Forfeited 清零）等。

### 1.2 最小不可违约束（A1–A8）

无仲裁/仅金额/公共时钟/拉取式结算/有界重组/可验证动作/争议期不可延长等（详见 nas_spec）。部署方提供 T_reorg 估计，并保证 T_dis ≥ 2·T_reorg。

---

## 2. 核心定理与证明素描（来自 nas_game_theory）

### 2.1 定理 1：SPNE 存在

有限期+完美信息→Kuhn 回溯归纳得到子博弈完美均衡；终态效用良定义（结清/没收）。

### 2.2 定理 2：活性（有限步终止）

以剩余时间为势函数，随时间单调减小；全局上界为 T_due+T_rev+T_dis。

### 2.3 定理 3：阈值均衡（A≤V+E_c）

争议态买方接受价的阈值满足 A≤V+E_c；∂/∂V>0、∂/∂E_c>0；时间项敏感性留待信息性回归（MET.5）。

### 2.4 定理 4：对称整蛊边界（conditional）

在对称托管与机会成本近似对称时，整蛊因子 G≤1；若条件不满足，G 可能>1，但受无量纲上界约束。

### 2.5 定理 5：变换式约束最优性（T1–T5）

T1 去分期；T2 对称化；T3 公共超时；T4 仅金额结算；T5 预先协商。逐步变换保持（甚至改进）最坏情形损失 L，传递性得到 NAS 对可行机制类的支配性（在 A1–A8 约束下）。

---

## 3. 比较静态与敏感性（信息性）

- 阈值：A≤V+E_c；提高 E_c 单调提升接受阈值；关于 T_dis 的符号取决于机会成本与协商成功率的权衡。  
- 运行敏感性：长周期争议（大 T_dis）与跨域 T_reorg 误设在运营上需要监测尾部行为；不改变定理与判据。

---

## 4. 判据与覆盖（唯一通过判据）

### 4.1 覆盖范围

本章判据仅覆盖 NAS 交易事件集合；比较表/决策树等为信息性，不纳入通过判据集。

### 4.2 判据与记录

- MET.4：整蛊因子 G 的目标统计量为 P50 ≤ 1.2（唯一通过判据）。
- MET.5：阈值回归（price ~ E_c + T_dis）为信息性记录，保存 R²与符号方向；不作为通过判据。

---

## 5. 复现与抽样（统一口径）

1) 观察窗口：24h；如 n<200 则扩至 7d 或直至 n≥200。  
2) 抽样方法：连续时间窗口全量；如总量过大，系统抽样步长 k=⌈N/200⌉（N=窗口内 NAS 订单总数）。  
3) 单位统一：时间字段（含 T_dis）统一“秒”；统计以事件日志为准。  
4) 工件标识：合约地址、观察窗口起止区块、参数变更交易哈希（如有）、告警触发/通知时间戳与通知渠道标识（优先级：消息哈希 > 渠道ID > 渠道名；介质=邮件/IM/工单之一）。

---

## 6. 基线与界（信息性）

- 直接支付：事后效率不可实现（Myerson–Satterthwaite 的特例直觉）。
- 中心化托管+仲裁：成本构成与路径依赖；非判据，仅作现实对照。
- HTLC/状态通道：适用于客观可验证/重复交互；不适于陌生方的主观性交付。

---

## 7. 实现与安全（信息性）

### 7.1 签名定价接口（EIP‑712/1271）

- 签名域（建议最小集）：`chainId, orderId, amountToSeller, proposer, nonce, deadline`。  
- 撤销/覆盖：无“链上撤销”；使用“新要约 + nonce 递增 + deadline 过期”。
- 事件字段（最小集）：`orderId, proposer, acceptor, amountToSeller, ts`。

### 7.2 安全基线

nonReentrant + CEI；入口前抢占 timeout*；失败重试与去重（≤3 行流程）。

### 7.3 L2/跨域适配

部署方给出 T_reorg 估计与执行域配置；白皮书仅信息性说明。

---

## 8. 扩展与稳健性（信息性）

- 比例路径（INV.9）：实现可选，不改变主结果；保持 A≤E_c 口径与 Pull 语义。  
- 异质主体与重复博弈：主结果在单次博弈与风险中性口径下成立；扩展分析留待研究与实证（不改变判据）。

---

## 9. 结语

NAS 在 A1–A8 约束下，以简单、可审计、可证明的机制达成可问责与可结算；用唯一判据（MET.4 P50≤1.2）保证运营可判定性，并将其余观测/对照/治理保持为信息性流程，符合 Minimal Enshrinement 与 Credible Neutrality 的原则。

---

## 附录 A：符号与术语

V/C/E_c/A/T_*，事件与状态、MET/GOV 指标，单位（时间统一“秒”）。

---

## 附录 B：Trace 示例

E14（Disputing→Settled，acceptPriceBySig）→ API: `acceptPriceBySig` → INV.2 → EVT: `PriceSettled/Settled` → MET: 报价接受率。

---

## 附录 C：参考与语义来源

- 规范（SSOT）：docs/zh/nas_spec.md  
- 博弈与测量：docs/zh/nas_game_theory.md

