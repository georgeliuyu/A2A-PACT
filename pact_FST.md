# PACT 状态博弈分析（归档版｜用于对照）

> 说明：本文件为“FST 归档稿”，用于与正式规范对照与历史比较。当前唯一生效规范请参见 `pact_spec.md`（正式颁布版）。本文不作为规范，仅保留原有分析结构与术语口径。

---

## 1. 模型与记号

- 参与者：$\mathsf{C}$（Client），$\mathsf{S}$（Contractor）
- 价值与成本：产品（或服务）价值 $V>0$，承包成本 $C>0$。
  - 过程口径：将 $C$ 理解为“当前剩余完成成本” $R(t)\ge 0$；已投入部分视为沉没成本 $I(t)\ge 0$。
- 时间与口径：
  - 可用性 $\kappa(t) \in [0,1]$；分阶段：$\kappa(\text{Executing}) \in [0,1]$，$\kappa(\text{Reviewing})=\kappa(\text{Disputing})=1$。
  - 托管 $E_c(t)$ 单调不减；承包方押金 $E_s=0$。
  - 计时器：`T_due, T_rev, T_dis` 为相对时长（单位：秒），各自的锚点分别为 `startTime`、`readyAt`、`disputeStart`；仅允许单调延长。
  - 时间戳：`startTime`（E1 设置）、`readyAt`（E3 设置）、`disputeStart`（E5/E11 设置）。
- 实际结清额：$A$ 为最终支付金额，满足 $0\le A \le E_c(t)$。
- 终态与收益（相对现金流）：
  - Settled：$(u_C, u_S)=(V-A,\;A-C)$；未支付部分 $\text{escrow}-A$ 返还 Client。
  - Forfeited：$(-E_c(t),\;0)$；Cancelled：$(0,0)$。

### 1.1 决策比较与边界

- 买家“付款 vs 没收”：$u_C(\text{pay})-u_C(\text{forfeit}) = E_c(t)-A \ge 0$；当 $A<E_c$ 时严格 >0。
- 卖家“继续/提交 vs 立即没收”：$u_S(\text{ready})-u_S(\text{forfeit}) = A-R(t) + I(t) \ge A-R(t)$；若 $R(t)<A$，继续/提交严格优于没收。
- 延时动作的理性边界：时间成本 $d_i(t)>0$ 使单边延长为自损。

---

## 2. 动作语义与时序要点

- acceptOrder：承包方接受订单，进入执行（副作用：`startTime=now`）。
- cancelOrder：取消订单（初始化阶段双向允许；执行阶段 Client 在“从未 Ready 且到期”下允许；Contractor 可随时取消）。
- markReady：承包方标记完成，进入评审；设置 `readyAt` 并起算 `T_rev`。
- approveReceipt：Client 确认结清（执行或评审阶段）。
- raiseDispute：任一方发起争议（执行或评审阶段），设置 `disputeStart` 并起算 `T_dis`。
- acceptPriceBySig：对手方在线验证离线报价并一次性结清（争议阶段）。
- timeoutSettle：评审超时自动结清。
- timeoutForfeit：争议超时自动没收。
- requestForfeit：即时没收（任一阶段即可触发）。
- withdrawPayout/withdrawRefund：提现（Pull 结算）。
- extendDue/extendReview/topUpEscrow：状态不变动作（仅延后/单调增加）。

---

## 3. 每个状态的博弈分析（表格式）

### 3.1 Initialized
| 动作 | 发起方 | 守卫/前置条件 | 转移 |
|---|---|---|---|
| acceptOrder(orderId) | Contractor | — | → Executing（副作用：`startTime=now`） |
| cancelOrder(orderId) | Client/Contractor | — | → Cancelled |
| topUpEscrow(orderId, amount) | Client | — | 状态不变 |

结论：进入 Executing 是主路径。

### 3.2 Executing
| 动作 | 发起方 | 守卫/前置条件 | 转移 |
|---|---|---|---|
| markReady(orderId) | Contractor | `now < startTime + T_due` | → Reviewing（重置 `T_rev`） |
| approveReceipt(orderId) | Client | — | → Settled |
| extendDue(orderId,newDueTs) | Client | `newDueTs > T_due` | 状态不变 |
| raiseDispute(orderId,reason) | Client/Contractor | — | → Disputing（`disputeStart=now`） |
| requestForfeit(orderId,reason) | Client/Contractor | — | → Forfeited |
| cancelOrder(orderId) | Client | 从未 Ready 且 `now ≥ startTime + T_due` | → Cancelled |
| cancelOrder(orderId) | Contractor | — | → Cancelled |
| topUpEscrow(orderId, amount) | Client | — | 状态不变 |

要点：理性路径是“及时 Ready → 付款或进入评审”。

### 3.3 Reviewing
| 动作 | 发起方 | 守卫/前置条件 | 转移 |
|---|---|---|---|
| approveReceipt(orderId) | Client | — | → Settled |
| raiseDispute(orderId,reason) | Client/Contractor | — | → Disputing（`disputeStart=now`） |
| requestForfeit(orderId,reason) | Client/Contractor | — | → Forfeited |
| timeoutSettle(orderId) | 任何 | 非争议且 `now ≥ readyAt + T_rev` | → Settled |
| extendReview(orderId,newRevSec) | Contractor | `newRevSec > T_rev` | 状态不变 |
| cancelOrder(orderId) | Contractor | — | → Cancelled |
| topUpEscrow(orderId, amount) | Client | — | 状态不变 |

要点：确认或开争议是理性选择；沉默将自动结清。

### 3.4 Disputing
| 动作 | 发起方 | 守卫/前置条件 | 转移 |
|---|---|---|---|
| acceptPriceBySig(orderId, amountToSeller, proposer, sig) | 对手方 | 验 EIP‑712/1271、nonce、deadline；`amountToSeller ≤ escrow` | → Settled |
| requestForfeit(orderId,reason) | Client/Contractor | — | → Forfeited |
| timeoutForfeit(orderId) | 任何 | `now ≥ disputeStart + T_dis` | → Forfeited |

要点：仅就“金额”协商；谈不拢由硬截止或没收终止。

### 3.5 Settled / 3.6 Forfeited / 3.7 Cancelled
- Settled：`withdrawPayout/withdrawRefund` 提现（Pull），状态保持。
- Forfeited：终态，无进一步动作。
- Cancelled：终态，无进一步动作。

---

## 4. 极端动作与可达性
- 卖方即时没收：在 $R(t) < A$ 边界下为严格劣策略。
- 买方即时没收：因 $A \le E_c(t)$，付款至少不劣；现实中 $d_C(t)>0$ 进一步推动付款。
- 对称均衡：极端动作可达但非优，形成稳定威慑。

---

## 5. 均衡与参数条件（提纲）
- 若存在 $A \in (C, \min(V, E_c(t_{settle}))]$ 且 $\kappa(\text{Review/Dispute})=1$，则“交付并结清”为子博弈完美均衡路径（SPE）。

---

## 6. 出金逻辑（Pull）与安全

- 入金：push；出金：pull。状态变更时仅“记账可领额”，实际转账在独立提现中完成。
- 结算场景：
  - Settled：`amountToSeller = escrow`（approve/timeout）或 `A`（协商）；退款 `refundToBuyer = escrow - amountToSeller`。
  - Cancelled：退款给 Client。
  - Forfeited：`escrow → ForfeitPool`，应得清零。
- 安全要点：
  - `ReentrancyGuard` + CEI；`SafeERC20` + 余额差核验；
  - 外部入口前抢占：若超时则优先处理没收/结清；
  - 单次支付不变量与幂等标记。

---

## 7. 可输出数据（链下，Informative）
- 终态分布（成功/没收/取消）；结清延迟；报价接受率；$A/E_c$ 基线；Top‑up 轨迹；极端动作发生率等。
