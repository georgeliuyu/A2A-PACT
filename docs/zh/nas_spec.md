# NAS 协议规范（A2A‑NAS，简称 NAS 协议）

别名：曾用名 PACT；本规范为 SSOT

发布状态：正式颁布版（Release）
版本：0.1
发布日期：2025-09-30

> 共识要点：本规范为唯一语义源（SSOT）。金额口径统一为 A ≤ E(t)，协议中无独立 P。以下清单为四位作者对齐后的统一结构与约束。
> 重要：`E(t)`、`D_due`、`D_rev`、`D_dis` 为“每笔订单协商/设定的参数或状态量”，协议层不提供固定默认值；客户端/产品层可给出模板或建议值，但不具规范效力。

## 0) 规范与追溯锚点
- 规范用语：MUST/SHOULD/MAY/MUST NOT。
- 语义优先级：本规范的“状态机与守卫” > 模型与记号 > 不变量 > API/事件 > 安全 > SLO > 治理数据 > 版本/变更 > 附录。
- ID 约定：
  - 状态转移边：E.x；不变量：INV.x；接口/事件：API.x/EVT.x；指标：MET.x（社会侧可用 GOV.x）；版本口径：VER.x；错误：ERR.x；结论：PR.x。
- Trace 原则（MUST）：每条 E.x 至少映射到一个 API/EVT，并覆盖相关 INV.x 与至少一项 MET.x/GOV.x。
- 发起方术语：client=买方，contractor=卖方，任意=任何 EOA 或合约地址（permissionless）。
- 术语大小写：client/contractor 与 Client/Contractor 等同（不区分大小写）。

## 1) 设计原则与非目标（Minimal Enshrinement）
- MUST NOT：将仲裁、信誉、治理投票等社会层逻辑内置到共识。
- MUST：协议仅提供可验证状态、事件与记账（Pull 结算）。
- MUST：金额口径仅以 A、E_c(t) 表达；禁用独立 P（不引入链上价格字段）。
- MUST：唯一机制：链下协商价格 → Client 上链托管等额资金；无争议路径全额结清，争议路径采用签名金额结清（见 §4）。
- MUST：资金仅经由 `depositEscrow` 单调增加；不支持减少或替换托管；订单创建不要求金额（escrow 初始为 0）。
- SHOULD：`acceptOrder` 不带最小金额参数；Contractor 仅在“托管=链下协商价”时调用接受（产品层闸门），实现层可选检查 `escrow > 0`。
- SHOULD：参数渐进、可观测驱动；保持实现简单、可审计。

注：NAS 的命名来源于 No‑Arbitration（无仲裁）原则。

## 2) 模型与记号（A ≤ E(t) 口径）
- 参与者：Client、Contractor。
- 时间与计时器：`startTime, readyAt, disputeStart`；`D_due, D_rev, D_dis`（相对时长，单位：秒；延长仅允许“向后”）。
- 资金与金额：`E(t)`（托管额，单调不减、不可减少）、`A`（实际结清额，0 ≤ A ≤ E(t_settle)）。
- 结清时间点定义（t_settle）：`approveReceipt` 确认时刻或 `timeoutSettle` 触发时刻，或 `acceptPriceBySig` 被对手方接受并链上确认的时刻；在该时刻计算 `A` 与退款。
- 可用性与成本：`κ(t)`（Executing∈[0,1]；Reviewing/Disputing=1）、`R(t)`、`I(t)`；收益 `(V−A, A−C)`。
- 对手与信息结构：自利/半理性；公开 `state/times/escrow`；协商采用可验证签名（EIP‑712/1271）。
- 单位与时间：金额以 token 最小单位计价；时间以 `block.timestamp`（秒）计；区块时间抖动不作为纠纷判断依据；命名约定：
  - `*Sec` 后缀仅用于 API/输入参数的相对时长（秒）（如 `newDueSec/newRevSec`）；API 参数命名 MUST 使用 `*Sec` 后缀；
  - 模型参数使用 `D_*` 命名（如 `D_due/D_rev/D_dis`），表示相对时长（秒）（MUST）；
  - `startTime/readyAt/disputeStart` 为绝对时间（时间戳）。
- 记号约定：文中 `now ≡ block.timestamp`；CEI ≡ checks–effects–interactions。

- 资产与代币（MUST）：
  - `tokenAddr` 表示资产标识（ERC‑20 合约地址或“原生 ETH”哨兵标识）。
  - 原生 ETH 支持：当订单资产为 ETH 时，`depositEscrow`/`createAndDeposit` 为 `payable`，要求 `msg.value == amount`；提现使用 `call` 且 `nonReentrant`。
  - ERC‑20 资产：要求 `msg.value == 0`，并使用 SafeERC20 `transferFrom` 扣划 `amount`（或在 `createAndDeposit` 中一并完成）。
  - 建议（SHOULD）：部署方可提供“封装 ETH（WETH）适配层”作为工程选项，但规范层面必须支持原生 ETH。

### 2.1 参数协商与范围（规范性）
- 协商主体与生效时点（MUST）：`E(t)`、`D_due`、`D_rev`、`D_dis` 由 Client 与 Contractor 针对“每一笔订单”达成一致；实现必须在订单建立/接受时将其作为“订单字段”固化存储。协议不提供全局/合约级默认值。
- 修改规则（MUST）：
  - `E(t)` 仅可通过 `depositEscrow` 单调增加，不得减少；
  - `D_due` 与 `D_rev` 仅允许在争议发生前“单调延后”（见 SIA1/SIA2），不得缩短；
  - `D_dis` 自设置后固定，协议层不提供延长入口（禁止 `extendDispute`）。
- 有界性（MUST）：三者必须为有限值且大于 0；为抵御链上重组，`D_dis ≥ 2·T_reorg`（由部署方按目标链给出 `T_reorg` 估计）。
- 无默认（MUST）：任何“默认/示例数值”（例如白皮书或工具链中的 `{E=500, D_rev=24h, D_dis=2h}`）均为信息性（informative），不具规范效力；实现不得在未显式传入的情况下暗设数值。
- 建议（SHOULD）：产品/客户端可根据场景提供“参数模板/推荐区间”，但应清晰标注为建议且允许用户覆盖。

## 3) 状态机与守卫（规范性，SSOT）

### 3.0 允许的转换（除此之外无其他，MUST）
注：变量与时间相关的副作用统一见 3.2 “守卫与副作用”。下列每条转移标注发起方：client、contractor、任意（三类含义：买方、卖方、无主体限制）。
- E1 Initialized -acceptOrder-> Executing（发起方：contractor）
- E2 Initialized -cancelOrder-> Cancelled（发起方：client/contractor）
- E3 Executing -markReady-> Reviewing（发起方：contractor）
- E4 Executing -approveReceipt-> Settled（发起方：client）
- E5 Executing -raiseDispute-> Disputing（发起方：client/contractor）
- E6 Executing -cancelOrder-> Cancelled（发起方：client）
- E7 Executing -cancelOrder-> Cancelled（发起方：contractor）
- E8 Executing -requestForfeit-> Forfeited（发起方：client/contractor）
- E9 Reviewing -approveReceipt-> Settled（发起方：client）
- E10 Reviewing -timeoutSettle-> Settled（发起方：任意）
- E11 Reviewing -raiseDispute-> Disputing（发起方：client/contractor）
- E12 Reviewing -cancelOrder-> Cancelled（发起方：contractor）
- E13 Reviewing -requestForfeit-> Forfeited（发起方：client/contractor）
- E14 Disputing -acceptPriceBySig-> Settled（发起方：对手方（client 或 contractor 之一））
- E15 Disputing -timeoutForfeit-> Forfeited（发起方：任意）
- E16 Disputing -requestForfeit-> Forfeited（发起方：client/contractor）

### 3.1 状态不变动作（SIA，MUST）
- SIA1：`extendDue(orderId, newDueSec)`（仅 client）要求 `newDueSec > 当前 T_due`（严格延后）。
- SIA2：`extendReview(orderId, newRevSec)`（仅 contractor）要求 `newRevSec > 当前 T_rev`（严格延后，按时长口径）。
- SIA3：`depositEscrow(orderId, amount)`（`payable`）要求 `amount > 0`，且 `escrow ← escrow + amount`（单调增加）。
  - 若订单资产为原生 ETH：MUST `msg.value == amount`；
  - 若为 ERC‑20：MUST `msg.value == 0`，并使用 SafeERC20 `transferFrom(client, this, amount)` 成功后记账。
- 适用范围：SIA3 允许于 Initialized/Executing/Reviewing；在 Disputing 及任何终态禁止充值（Top‑up）。

### 3.2 守卫与副作用（MUST）
- 参数持久化（MUST）：实现必须在订单建立/接受时持久化记录 `D_due/D_rev/D_dis` 的初值（来自双方协商）；不得在链上以“隐式默认”替代缺失值。
- startTime/readyAt/disputeStart 为一次性锚点（设置后 MUST NOT 回拨或重置）；`D_due/D_rev` 仅允许单调延长；不提供 extendDispute。
- G.E1：`acceptOrder` 仅当 `state=Initialized`；副作用：`startTime = now`（首次进入 Executing 时设置锚点）。
- G.E3：`markReady` 仅当 `now < startTime + D_due`；副作用：设置 `readyAt=now` 并（重新）起算评审计时 `D_rev`。
- G.E4/E9：`approveReceipt` 仅适用于 `state ∈ {Executing, Reviewing}`（Disputing 不适用）。
- G.E10：`timeoutSettle` 仅当 `state=Reviewing` 且 `now ≥ readyAt + D_rev`。
- G.E5/E11：`raiseDispute` 允许于 Executing/Reviewing 任意时刻进入 Disputing；副作用：设置 `disputeStart=now`。
- G.E6：`cancelOrder`（client）仅当“从未 Ready（`readyAt` 未设置）且 `now ≥ startTime + D_due`”。
- G.E7：`cancelOrder`（contractor）允许（无额外守卫）。
- G.E14：`acceptPriceBySig` 仅当 `state=Disputing`，且 `amountToSeller ≤ escrow` 并通过 EIP‑712/1271 签名、nonce、deadline 校验。
- G.E15：`timeoutForfeit` 仅当 `state=Disputing` 且 `now ≥ disputeStart + D_dis`。

### 3.3 终态约束（MUST）
- `Settled/Forfeited/Cancelled` 为终态；到达终态后不得再改变状态或资金记账；仅允许提现型入口读取并领取既有可领额（若有）。

## 4) 结算与不变量（Pull 支付）
- 金额计算
  - INV.1 全额结清：`amountToSeller = escrow`（approve/timeout）。
  - INV.2 金额型结清：`amountToSeller = A` 且 `0 ≤ A ≤ escrow`（签名协商）。
  - INV.3 退款：`refundToBuyer = escrow − amountToSeller`（若 A < escrow）。
- 资金安全
  - INV.4 单次支付：每单至多一次成功卖方提现（single_payout）。
  - INV.5 幂等标记：提现成功后标记，重复调用无副作用。
  - INV.6 入口前抢占：外部入口先处理 `timeout*`，防延迟攻击。
  - INV.7 资产与对账：SafeERC20 + 余额差核验；必要时采用白名单/适配层。
- 资金去向与兼容
  - INV.8 Forfeited：`escrow → ForfeitPool`（不向外部分配），`owed/refund` 清零。
  - INV.9 比例路径兼容（可选）：`amountToSeller = floor(escrow * num / den)`；余数全部计入买方退款。实现 MUST 使用安全的“mulDiv 向下取整”或等效无溢出实现；任何溢出/下溢 MUST revert；禁止四舍五入与精度提升。
- INV.10 Pull 语义：状态变更仅“记账可领额”，实际转账仅在 `withdrawPayout/withdrawRefund` 发生；禁止在状态变更入口直接 `transfer`。
 - INV.11 锚点一次性：`startTime/readyAt/disputeStart` 一旦设置，MUST NOT 修改或回拨（仅允许“未设置 → 设置一次”）。
  - INV.12 计时器规则：`D_due/D_rev` 仅允许延后（单调增加，且在进入 Disputing 前）；`D_dis` 固定且不可延长。
  - INV.13 唯一机制：无争议路径必须全额结清；争议路径采用签名金额结清；金额口径始终满足 `A ≤ E_c(t_settle)`，链上不出现价格字段。

## 5) 安全与威胁模型
- 签名与重放（MUST）：采用 EIP‑712/1271；签名数据至少包含 `{orderId, amountToSeller(=A), proposer, nonce, deadline}`；`amountToSeller ≤ escrow`；`nonce` 的作用域至少为 `{orderId, proposer}` 且一次性消费；`deadline` 基于 `block.timestamp` 判定；必须防止跨订单重放。
- 重入与交互顺序：提现 `nonReentrant`，遵循 CEI。
- 时间边界：统一区块时间；`D_due/D_rev` 仅允许延后（单调），`D_dis` 固定且不可延长。
- 残余风险：破坏型对手导致的没收外部性 → 由社会层资质/信誉/稽核约束。

## 6) 收益比较与均衡（SPE 概要）
- PR.1 付款不劣：`u_C(pay) − u_C(forfeit) = E(t) − A ≥ 0`；当 `A<E(t)` 时严格 > 0。
- PR.2 卖方即刻没收为劣：若 `R(t) < A`，则 `A − R(t) > 0 ≥ −I(t)`。
- PR.3 SPE（充分条件）：存在 `A ∈ (C, min(V, E(t_settle))]` 且 `κ(Review/Dispute)=1`，则“交付并结清”为子博弈完美均衡路径。
- PR.4 比较静态：Top‑up 单调把买方偏好由 `forfeit/tie` 推向 `pay`。

## 7) API 与事件契约（映射状态机/不变量）
- 函数（最小集）：
  - `createOrder(tokenAddr, contractor, dueSec, revSec, disSec) -> orderId`
  - `createAndDeposit(tokenAddr, contractor, dueSec, revSec, disSec, amount)`（payable）→ 创建并充值（ETH: `msg.value==amount`；ERC‑20: `transferFrom` amount）
  - `depositEscrow(orderId, amount)`（payable；同上资产规则）
  - `acceptOrder(orderId)`；`markReady(orderId)`；`approveReceipt(orderId)`；`timeoutSettle(orderId)`
  - `raiseDispute(orderId)`；`acceptPriceBySig(orderId, payload, sig1, sig2)`；`timeoutForfeit(orderId)`
  - `cancelOrder(orderId)`；`withdrawPayout(orderId)`；`withdrawRefund(orderId)`
  - `extendDue(orderId, newDueSec)`；`extendReview(orderId, newRevSec)`（单调延后）
- 事件（建议的最小充分字段）：
  - `OrderCreated(orderId, client, contractor, tokenAddr, dueSec, revSec, disSec, ts)`
  - `EscrowDeposited(orderId, from, amount, newEscrow, ts)`；`Accepted(orderId, escrowAtAccept, ts)`
  - `Settled(orderId, amountToSeller, escrowAtSettle, ts, actor)`（`actor ∈ {Client, Timeout}`）
  - `PriceSettled(orderId, proposer, acceptor, amountToSeller, nonce, ts)`
  - `Forfeited(orderId, ts)`；`Cancelled(orderId, ts, cancelledBy)`（`cancelledBy ∈ {Client, Contractor}`）
  - `PayoutWithdrawn(orderId, to, amount, ts)`；`RefundWithdrawn(orderId, to, amount, ts)`
- 错误（示例）：ERR.1 `ErrInvalidState`；ERR.2 `ErrGuardFailed`；ERR.3 `ErrAlreadyPaid`；ERR.4 `ErrExpired`；ERR.5 `ErrBadSig`；ERR.6 `ErrOverEscrow`。
- 映射规则（MUST）：E.x ↔ API/EVT ↔ INV.x ↔ MET.x/GOV.x 一一可追溯。
- 事件最小字段（建议）：
  - `EscrowDeposited(orderId, from, amount, newEscrow, ts)`；
  - `Accepted(orderId, escrowAtAccept, ts)`；
  - `Settled(orderId, amountToSeller, escrowAtSettle, ts, actor)`，其中 `actor ∈ {Client, Timeout}`；
  - `PriceSettled(orderId, proposer, acceptor, amountToSeller, nonce, ts)`；
  - `Forfeited(orderId, ts)`；`Cancelled(orderId, ts, cancelledBy)`，其中 `cancelledBy ∈ {Client, Contractor}`；
  - `PayoutWithdrawn(orderId, to, amount, ts)`；`RefundWithdrawn(orderId, to, amount, ts)`。
- （工程建议）资产支持与适配层细节见平台工程文档。

## 8) 可观测性与 SLO
- 指标（示例）：
  - MET.1 结清延迟 P95、超时触发率；
  - MET.2 提现失败率、重试率；
  - MET.3 资金滞留余额；
  - MET.4 争议时长分布、报价接受率；
  - MET.6 状态转换延迟/吞吐（E1/E3/E5/E7/E12 等非终态转换的时延与速率）；
  - GOV.1 终态分布（成功/没收/取消）；
- GOV.2 `A/E` 基线分布（基线 `E^ready` 或 `E^dispute`，见 VER）。
- SLO（示例）：提现成功率 ≥ 99.9%；结清到账 P50 < 1 区块、P95 < 3 区块；资金滞留余额 < 0.1%（周）。
- 熔断/回滚：超阈触发“切换白名单/停写/回滚”剧本（与第10、11章联动）。

## 9) 治理与数据（社会层）
- 口径版本化（VER）：明确窗口（如 7 天）、基线定义（`E^ready`/`E^dispute`）、字段来源与版本号。
- 隐私与可解释：最小披露、可验证日志；保留 `nonce/deadline/proposer` 证据链；提供申诉/审计通道（模板化）。

## 10) 版本化与变更管理
- 语义版本：状态机/INV/API/MET 任一变更 → 次版本以上；破坏性变更 → 主版本。
- 变更卡（CHG）：记录影响面（4→5→7→8 链路）、迁移/回滚步骤、兼容窗口。

## 11) 渐进路线图
- T0（MVP）：状态机完整、Pull 结算、不变量与基础指标；
- T1：批量提现、事件驱动自动领取、最小代付（4337/2771）；
- T2：非常规代币适配层与白名单策略、跨域回退工具；
- T3：接口标准化与生态协作，保持“最小内置”。

## 12) 附录（追溯矩阵）
风格说明：本节“转换箭头”的规范写法为 ASCII `->`；如渲染为 Unicode 箭头（→），视为等价显示，不改变含义。
信息性示例（Trace）：
- Trace.1：E4（Executing->Settled，approveReceipt） -> API: `approveReceipt` -> INV.1 -> EVT: `Settled` -> MET: 结清延迟/资金滞留。
- Trace.2：E14（Disputing->Settled，acceptPriceBySig） -> API: `acceptPriceBySig` -> INV.2 -> EVT: `PriceSettled, Settled` -> MET: 争议时长/报价接受率。
- Trace.3：E15（Disputing->Forfeited，timeoutForfeit） -> API: `timeoutForfeit` -> EVT: `Forfeited` -> GOV: 没收率/争议时长。

全量映射（覆盖 E1..E16；至少一项指标）：
- E1 Initialized->Executing（acceptOrder） -> API: `acceptOrder` -> INV: INV.11 -> EVT: (none) -> MET: MET.6（接单延迟）。
- E2 Initialized->Cancelled（cancel） -> API: `cancelOrder` -> INV: INV.3（退款计算） -> EVT: `Cancelled` -> GOV: GOV.1。
- E3 Executing->Reviewing（markReady） -> API: `markReady` -> INV: INV.11 -> EVT: (none) -> MET: MET.6（执行->评审时延）。
- E4 Executing->Settled（approveReceipt） -> API: `approveReceipt` -> INV.1 -> EVT: `Settled` -> MET: MET.1/MET.3。
- E5 Executing->Disputing（raiseDispute） -> API: `raiseDispute` -> INV: INV.11 -> EVT: (none) -> MET: MET.6（争议触发时延）。
- E6 Executing->Cancelled（cancel: Client 条件） -> API: `cancelOrder` -> INV: INV.3 -> EVT: `Cancelled` -> GOV: GOV.1。
- E7 Executing->Cancelled（cancel: Contractor） -> API: `cancelOrder` -> INV: INV.3 -> EVT: `Cancelled` -> GOV: GOV.1。
- E8 Executing->Forfeited（requestForfeit） -> API: `requestForfeit` -> INV: INV.8 -> EVT: `Forfeited` -> GOV: GOV.1。
- E9 Reviewing->Settled（approveReceipt） -> API: `approveReceipt` -> INV.1 -> EVT: `Settled` -> MET: MET.1/MET.3。
- E10 Reviewing->Settled（timeoutSettle） -> API: `timeoutSettle` -> INV: INV.1 -> EVT: `Settled` -> MET: MET.1。
- E11 Reviewing->Disputing（raiseDispute） -> API: `raiseDispute` -> INV: INV.11 -> EVT: (none) -> MET: MET.6（争议触发时延）。
- E12 Reviewing->Cancelled（cancel: Contractor） -> API: `cancelOrder` -> INV: INV.3 -> EVT: `Cancelled` -> GOV: GOV.1。
- E13 Reviewing->Forfeited（requestForfeit） -> API: `requestForfeit` -> INV: INV.8 -> EVT: `Forfeited` -> GOV: GOV.1。
- E14 Disputing->Settled（acceptPriceBySig） -> API: `acceptPriceBySig` -> INV: INV.2 -> EVT: `PriceSettled, Settled` -> MET: MET.4（报价接受率）。
- E15 Disputing->Forfeited（timeoutForfeit） -> API: `timeoutForfeit` -> INV: INV.8 -> EVT: `Forfeited` -> GOV: GOV.1/GOV.3（争议时长）。
- E16 Disputing->Forfeited（requestForfeit） -> API: `requestForfeit` -> INV: INV.8 -> EVT: `Forfeited` -> GOV: GOV.1。

---
