# NAS 协议唯一机制规范（spec_new）

别名：A2A‑NAS（唯一机制版）｜曾用名 PACT（Preemptive Accountability via Escrow）

发布状态：Draft（草案，用于收敛“唯一机制”）
版本：0.1
发布日期：2025‑10‑05

提要（Summary）
- 唯一机制：链下协商价格 → client 上链托管同额资金（仅允许增加）→ 无争议全额结清；若需降价或出现分歧，则进入争议并用签名金额结清。
- 链上不引入独立价格变量 P；仅使用结清额 A 与托管额 E_c(t)，且始终满足 A ≤ E_c(t_settle)。
- `depositEscrow` 是唯一资金入口（首充/追加同函数）；`acceptOrder` 不带 `minEscrow`；contractor 仅在看到“托管=链下价”时调用接受。

注：本规范为“唯一机制版”草案，用于替代多机制分叉；如与既有 nas_spec.md 存在措辞差异，以本文件描述的强约束为准（合并后更新 SSOT 标注）。

## 0) 术语与记号（规范性）
- 参与方：Client（买方）、Contractor（卖方）。
- 金额与时间：
  - `E_c(t)`：当前托管额（escrow），单调不减、不可减少。
  - `A`：结清金额，结清时刻需满足 `0 ≤ A ≤ E_c(t_settle)`。
  - `startTime, readyAt, disputeStart`：绝对时间锚点（区块时间，秒）。
  - `T_due, T_rev, T_dis`：相对时长（秒）；每单协商，协议不提供默认值；`T_dis` 固定不可延长。
- 状态：Initialized, Executing, Reviewing, Disputing, Settled, Forfeited, Cancelled。
- 规范用语：MUST/SHOULD/MAY/MUST NOT 采用 RFC2119 语义。

## 1) 设计原则（唯一机制）
- MUST：链上无独立价格字段 P；金额口径仅以 A 与 E_c(t) 表达。
- MUST：除本节所述路径外，不存在其他价格/结清机制。
- MUST：无争议路径“全额结清”（approveReceipt/timeoutSettle → `amountToSeller = E_c`）。
- MUST：争议路径“签名金额结清”（acceptPriceBySig → `amountToSeller = A` 且 `A ≤ E_c`），差额退款给买方。
- MUST：`depositEscrow` 为唯一资金入口；仅允许增加，不允许减少或替换。
- MUST：`T_due/T_rev/T_dis` 为每单协商参数，在订单建立时固化；`T_due/T_rev` 仅允许单调延后，`T_dis` 固定不可延长。
- SHOULD：产品层仅在“托管=链下价”时暴露 `acceptOrder` 入口；否则不显示/不允许。

## 2) 状态机与转换（规范性）
允许的转换仅限于下列列表（除此之外无其他）：
- E1 Initialized -acceptOrder-> Executing（发起：contractor）
- E2 Initialized/Executing/Reviewing -cancelOrder-> Cancelled（发起：client/contractor；各自守卫见 §3）
- E3 Executing -markReady-> Reviewing（发起：contractor）
- E4 Executing/Reviewing -approveReceipt-> Settled（发起：client）
- E5 Executing/Reviewing -raiseDispute-> Disputing（发起：client/contractor）
- E10 Reviewing -timeoutSettle-> Settled（发起：任意）
- E14 Disputing -acceptPriceBySig-> Settled（发起：对手方之一）
- E15 Disputing -timeoutForfeit-> Forfeited（发起：任意）

注：E2 的具体守卫区分见 §3（未就绪/超时等条件）。

## 3) 守卫与副作用（规范性）
- G.DEP：`depositEscrow(orderId, amount)`（仅 Client）
  - 要求：`amount > 0`；`state ∈ {Initialized, Executing, Reviewing}`。
  - 副作用：`escrow ← escrow + amount`（单调增加）。
  - 禁止：`state ∈ {Disputing, Settled, Forfeited, Cancelled}`。
- G.E1：`acceptOrder(orderId)`（仅 Contractor）
  - 要求：`state=Initialized`；（建议）`escrow > 0`。
  - 副作用：设置 `startTime = now`。
- G.E3：`markReady(orderId)`（仅 Contractor）
  - 要求：`state=Executing ∧ now < startTime + T_due`。
  - 副作用：设置 `readyAt = now`。
- G.E4：`approveReceipt(orderId)`（仅 Client）
  - 要求：`state ∈ {Executing, Reviewing}`。
  - 副作用：进入 Settled，并记账 `amountToSeller = escrow`（全额结清）。
- G.E10：`timeoutSettle(orderId)`（任意）
  - 要求：`state=Reviewing ∧ now ≥ readyAt + T_rev`。
  - 副作用：进入 Settled，并记账 `amountToSeller = escrow`（全额结清）。
- G.E5：`raiseDispute(orderId)`（Client 或 Contractor）
  - 要求：`state ∈ {Executing, Reviewing}`。
  - 副作用：进入 Disputing，设置 `disputeStart = now`。
- G.E14：`acceptPriceBySig(orderId, payload, sigC, sigR)`（对手方之一）
  - 要求：`state=Disputing` 且 `payload.amountToSeller = A ≤ escrow`。
  - 签名：EIP‑712/1271；`payload` 至少含 `{orderId, amountToSeller, proposer, acceptor, nonce, deadline}`；
    - `nonce` 的作用域至少 `{orderId, signer}` 且一次性消费；
    - `deadline` 基于 `block.timestamp` 判定；
    - 必须防跨订单/跨合约重放。
  - 副作用：进入 Settled 并记账 `amountToSeller = A`。
- G.E15：`timeoutForfeit(orderId)`（任意）
  - 要求：`state=Disputing ∧ now ≥ disputeStart + T_dis`。
  - 副作用：进入 Forfeited；`escrow → ForfeitPool`；清零可领额。
- 取消（`cancelOrder(orderId)`）
  - Client 取消（仅当“从未 Ready”且 `now ≥ startTime + T_due`）。
  - Contractor 取消（允许，状态需满足 `state ∈ {Initialized, Executing, Reviewing}` 且未进入 Disputing）。
  - 进入 Cancelled；退款按 §4 不变量计算。

锚点一次性（MUST）：`startTime/readyAt/disputeStart` 设置后不得修改或回拨。

## 4) 结算与不变量（规范性）
- INV.1 全额结清：`approveReceipt/timeoutSettle => amountToSeller = escrow`。
- INV.2 金额结清：`acceptPriceBySig => amountToSeller = A` 且 `0 ≤ A ≤ escrow`。
- INV.3 退款：`refundToBuyer = escrow − amountToSeller`（若 `A < escrow`）。
- INV.4 托管单调不减：仅 `depositEscrow` 可以增加托管；不提供减少/替换入口。
- INV.5 单次卖方提现与幂等：`withdrawPayout` 成功后标记，重复调用无副作用；退款同理。
- INV.6 入口前抢占：外部入口应优先处理 `timeout*`，防延迟攻击。
- INV.7 Pull 语义：状态变更仅“记账可领额”，真实转账仅在 `withdrawPayout/withdrawRefund` 发生；
  禁止在状态入口直接 `transfer`。

## 5) 接口（最小充分集，信息性签名）
- 创建与资金
  - `createOrder(token, contractor, T_dueSec, T_revSec, T_disSec)`（Client）：初始化订单（`escrow=0`）。
  - `depositEscrow(orderId, amount)`（Client）：首充/追加，`amount>0`；允许于 `Initialized/Executing/Reviewing`。
- 进度与结清
  - `acceptOrder(orderId)`（Contractor）：进入 Executing；设置 `startTime`。
  - `markReady(orderId)`（Contractor）：进入 Reviewing；设置 `readyAt`。
  - `approveReceipt(orderId)`（Client）：Settled，全额结清。
  - `timeoutSettle(orderId)`（任意）：Settled，全额结清（`readyAt+T_rev`）。
- 争议与没收
  - `raiseDispute(orderId)`（任意一方）：进入 Disputing；设置 `disputeStart`。
  - `acceptPriceBySig(orderId, payload, sig1, sig2)`：签名金额结清（`A ≤ escrow`）。
  - `timeoutForfeit(orderId)`（任意）：进入 Forfeited（`disputeStart+T_dis`）。
- 取消与提现
  - `cancelOrder(orderId)`（Client/Contractor，受 §3 守卫限制）。
  - `withdrawPayout(orderId)` / `withdrawRefund(orderId)`：卖方/买方在终态领取已记账可领额。
- 计时器调整（单调延后）
  - `extendDue(orderId, newDueSec)`（仅 Client，`newDueSec > 当前 T_due`）。
  - `extendReview(orderId, newRevSec)`（仅 Contractor，`newRevSec > 当前 T_rev`）。

## 6) 安全（规范性）
- 签名与重放：EIP‑712/1271；域隔离（链 ID、合约地址）、`orderId`、`nonce` 单次消费、`deadline` 检查；防跨订单/跨合约重放。
- 交互顺序：提现入口 `nonReentrant`；遵循 CEI（checks‑effects‑interactions）。
- 金额计算：若实现可选“比例路径”，必须使用安全 `mulDiv` 下取整并防溢出；禁止四舍五入提精度。
- 时间口径：统一使用 `block.timestamp`（秒）；区块抖动不作为纠纷判断依据。

## 7) 事件（最小充分）
- `EscrowDeposited(orderId, from, amount, newEscrow, ts)`
- `Accepted(orderId, escrowAtAccept, ts)`
- `Ready(orderId, escrowAtReady, ts)`（可选）
- `Settled(orderId, amountToSeller, escrowAtSettle, ts, actor)`（`actor ∈ {Client, Timeout}`）
- `PriceSettled(orderId, proposer, acceptor, amountToSeller, nonce, ts)`
- `Forfeited(orderId, ts)`
- `Cancelled(orderId, ts, by)`（`by ∈ {Client, Contractor}`）
- `PayoutWithdrawn(orderId, to, amount, ts)` / `RefundWithdrawn(orderId, to, amount, ts)`

## 8) 用户流（信息性）
- 固定价（常规）
  1) 链下约价 p → Client `depositEscrow=p`。
  2) Contractor `acceptOrder` 接单（仅当“托管=链下价”时才操作）。
  3) 交付 → `markReady` → Client `approveReceipt` 全额结清（=escrow）。
- 涨价
  1) 链下改为 p' → Client 先 `depositEscrow` 补至 `E_c≥p'`。
  2) 正常推进（`markReady/approve` 或超时结清）。
- 降价/部分结清
  1) 进入 Disputing → 双签 `acceptPriceBySig(A<p)`。
  2) 以 A 结清，差额 `escrow−A` 退款给 Client。

## 9) 与旧机制对比（设计动机）
- 无链上 P：减少攻击面与实现复杂度；唯一不变量 `A ≤ E_c` 易审计与形式化。
- 等额托管先行：提升可信承诺与激励相容；涨价须先补足托管，避免事后要挟。
- 收敛与活性：无争议全额结清；仅在争议中才协商 A；流程有限步终止。
- 对称威慑：配合 Forfeit 路径抑制“廉价整蛊”，保护弱势方。

## 10) 指标与运维（信息性）
- 指标示例：
  - 结清延迟（P50/P95）、超时率；
  - 报价接受率（PriceSettled 占比）；
  - 资金滞留余额、提现失败率；
  - 争议时长分布、`A/E_c` 分布（以 `E_c^ready` 或 `E_c^dispute` 为基线）。
- SLO 示例：提现成功率 ≥ 99.9%；结清到账 P95 < 3 区块；周度资金滞留 < 0.1%。

—— 规范正文完 ——

