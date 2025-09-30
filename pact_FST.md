# PACT 状态博弈分析（完整稿｜不改协议规则）

> 约束前提：**完全去中心化**；**承包方无押金** $E_s=0$；**客户托管额** $E_c(t)\ge0$（可上调、不可减少）；**可用性** $\kappa(state)$ 定义见下。
> 只做**博弈分析与可验证性质**，**不修改协议规则**；信誉体系不在本文设计，仅在文末列“可输出数据”。

---

## 1. 模型与记号

* 参与者：$\mathsf{C}$（Client），$\mathsf{S}$（Contractor）
* 价值与成本：产品（或者服务）价值 $V>0$，承包成本 $C>0$。
  * 为贴合过程型项目，后文将 $C$ 理解为“当前时点的剩余完成成本” $R(t)\ge0$（即从现在到完成仍需投入的成本；已投入部分视为沉没成本）。因此比较收益时以 $A-R(t)$（其中 $A\le E_c(t)$ 为实际结清额）替代固定价口径；当仅给出单一 $C$ 时可视作对 $R(t)$ 的上界近似。
  * 已发生成本 $I(t)\ge0$：截至时刻 $t$ 已投入且不可回收的“沉没成本”（sunk cost）。$I(t)$ 仅用于绝对收益记账，不进入“继续/放弃”的决策边界；常用近似关系为 $R(t)\approx C_{\text{total}}-I(t)$（其中 $C_{\text{total}}$ 为总成本上界）。
* **时间函数**：
  * $\kappa(t)$：可用性因子，取值 $[0,1]$，反映交付物对买家的"可长期使用"程度
    * 极端/上界分析：$\kappa(t)=1$（链下交付随时可能达到1，用作保守边界）
    * 按阶段口径：$\kappa(\text{Executing}) \in [0,1]$，$\kappa(\text{Reviewing})=\kappa(\text{Disputing})=1$
  * $E_c(t)$：买方托管额随时间的可变函数，单调不减（允许Top‑up但不减少）
  * $d_i(t) \geq 0$：主体 $i \in \{C,S\}$ 的时间成本/折现（机会成本、资金成本、气费、声誉），用于解释"单边延时=自损"的理性基础
* **时间戳/计时器**：
  * `startTime`：进入Executing的起点时间
  * `readyAt`：markReady的时间点；触发评审定时$T_{rev}$的基准
  * `disputeStart`：进入Disputing的时间点；触发没收定时$T_{dis}$的基准
  * $T_{due}$：交付到期时间（绝对时间戳，与`extendDue(newDueTs)`一致）
  * $T_{rev}$：评审窗口时长（从`readyAt`开始计时）
  * $T_{dis}$：争议窗口时长（从`disputeStart`开始计时）
* 托管：$E_c(t)\ge0$（单调不减），$E_s=0$
* 实际结清额：$A$ 为最终支付金额，满足 $0\le A \le E_c(t_{settle})$
* 终态与收益（相对合约现金流；绝对收益需再加 $I(t)$/机会成本）：

  * **Settled**：$(u_C,u_S)=(V-A,\;A-C)$，其中 $A\in[0,E_c(t_{settle})]$；未支付部分 $(\text{escrow}-A)$ 返还 Client
  * **Forfeited**：$(u_C,u_S)=(-E_c(t),\;0)$
  * **Cancelled**：$(0,0)$

* 本文收益为相对合约现金流的净额；若需绝对收益，再叠加机会成本 $I(t)$

### 1.1 统一收益与决策边界

**买家"付款 vs 没收"的比较**（任一时刻 $t$）：
- $u_C(\text{pay}) = \kappa(t)\cdot V - A$；$u_C(\text{forfeit}) = \kappa(t)\cdot V - E_c(t)$
- **结论**：由于 $A\le E_c(t)$，有 $u_C(\text{pay})\ge u_C(\text{forfeit})$；当 $A=E_c(t)$ 时打平，$A<E_c(t)$ 时付款严格更优。
- **默认**：未进入争议时，`approveReceipt` 视为“全额结清”：$A=E_c(t)$。
- **约束来源**：余额约束 $A\le \text{escrow}=E_c(t)$。

**卖家"继续做/提交 vs 立即没收"的比较**（任一时刻 $t$）：
- $u_S(\text{ready}) = A - R(t)$；$u_S(\text{forfeit}) = -I(t)-E_s$  
- **结论**：只要 $R(t) \leq A + E_s$，继续/提交弱优于立即没收；在本文假设 $E_s=0$ 下，$R(t)\le A$ 即成立，且默认 $A\approx E_c(t)$（未争议）。

**延时动作的理性边界**：
- 时间成本$d_i(t) > 0$使单边延长为自损动作（见第2节"允许但自损"设计原则）

---

## 2. 协议时序要点与动作速览

### 2.1 主要动作语义
| 动作 | 语义 |
|------|------|
| `acceptOrder` | 承包方接受订单，开始执行 |
| `cancelOrder` | 取消订单（条件见§3），退款/返还 |
| `markReady` | 承包方标记工作完成，进入评审期 |
| `approveReceipt` | 客户接受交付，正常结算 |
| `acceptPriceBySig` | 争议期协商价格结算 |
| `timeoutSettle` | 评审超时自动结算 |
| `timeoutForfeit` | 争议超时自动没收 |
| `withdrawPayout` | 承包方提取收益 |
| `withdrawRefund` | 客户提取退款（若有余额） |
| `raiseDispute` | 任一方发起争议 |
| `requestForfeit` | 立即没收（无需超时） |
| `extendDue` | 客户延长交付期限 |
| `extendReview` | 承包方延长评审期限 |
| `topUpEscrow` | 客户增加托管金额 |

*各状态的可用动作与守卫条件详见§3。*

**"允许但自损"的设计原则**：协议允许主体执行对自身不利的动作（如单边延时、立即没收），以保持去中心化与可达性；理性前提 + 时间成本使其在均衡外不被滥用。

### 2.2 状态转移图（FST）

```
  Initialized ── acceptOrder ──→ Executing
       │                         │
       └── cancel ───────────────┘

  Executing ── markReady ───────→ Reviewing
  Executing ── approveReceipt ──→ Settled
  Executing ── raiseDispute ────→ Disputing
  Executing ── cancel（Client 条件/Contractor） ─→ Cancelled
  Executing ── requestForfeit ──→ Forfeited

  Reviewing ── approveReceipt ─→ Settled
  Reviewing ── timeoutSettle ──→ Settled
  Reviewing ── raiseDispute ───→ Disputing
  Reviewing ── cancel（Contractor） ─→ Cancelled
  Reviewing ── requestForfeit ─→ Forfeited

  Disputing ── acceptPriceBySig ─→ Settled
  Disputing ── timeoutForfeit ───→ Forfeited
  Disputing ── requestForfeit ───→ Forfeited

  Settled/Forfeited/Cancelled：终态（提现不改变状态）
  注：extendDue/extendReview/topUpEscrow 为状态不变动作
```

---

## 3. 每个状态的博弈分析

### 3.1 Initialized

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| `acceptOrder(orderId)` | Contractor | — | → Executing |
| `cancelOrder(orderId)` | Client/Contractor | — | → Cancelled |
| `topUpEscrow(orderId, uint256 amount)` | Client | — | 状态不变 |

* **理性选择**：若存在可实现的结清额 $A$ 使得 $A>C$（保守可取 $A\approx E_c(\text{当前})$ 的下界），$\mathsf{S}$ 开工的期望收益为正；`cancel` 为非对抗性终止。
* **结论**：进入 Executing 是主路径。

### 3.2 Executing

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| `markReady(orderId)` | Contractor | `now < startTime + T_due` | → Reviewing（并重置 `T_rev`） |
| `approveReceipt(orderId)` | Client | — | → Settled |
| `extendDue(orderId,newDueTs)` | Client | 仅延后；`newDueTs > 当前 T_due` | 仅更新到期时间（状态不变） |
| `raiseDispute(orderId,reason)` | Client/Contractor | — | → Disputing |
| `requestForfeit(orderId,reason)` | Client/Contractor | — | → Forfeited |
| `cancelOrder(orderId)` | Client | 从未 Ready 且 `now ≥ startTime + T_due` | → Cancelled |
| `cancelOrder(orderId)` | Contractor | — | → Cancelled |
| `topUpEscrow(orderId, uint256 amount)` | Client | — | 状态不变 |
* **卖方视角（前瞻比较）**：

  * `markReady`→期望边际收益 $A-R(t)$（$A\le E_c(t)$；未争议阶段默认 $A=E_c(t)$）为正。
  * 极端（卖方即刻没收，对称）：绝对收益 $u_S^{\text{forfeit}}=-I(t)-E_s$、$u_S^{\text{ready}}=A-R(t)$；收益差 $\Delta_S=A-R(t)+E_s$。在 $E_s=0$ 时，若 $R(t)<A$，即刻没收为严格劣。

* **买方视角**：

  * 直接 `approveReceipt`→$\kappa(t)\cdot V - A$（默认 $A=E_c(t)$）。
  * （执行期仅强调卖方极端；买方极端见 3.3 Reviewing。）
* **结论**：理性路径是“及时 Ready → 付款或进入评审”。

### 3.3 Reviewing

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| `approveReceipt(orderId)` | Client | — | → Settled |
| `raiseDispute(orderId,reason)` | Client/Contractor | — | → Disputing |
| `requestForfeit(orderId,reason)` | Client/Contractor | — | → Forfeited |
| `timeoutSettle(orderId)` | 任何 | 非争议且 `now ≥ readyAt + T_rev` | → Settled（自动路径） |
| `extendReview(orderId,newRevSec)`| Contractor | 仅延后；`newRevSec > 当前 T_rev` | 仅更新评审时长（状态不变） |
| `cancelOrder(orderId)` | Contractor | — | → Cancelled |
| `topUpEscrow(orderId, uint256 amount)` | Client | — | 状态不变 |

* **买方**（见1.1的$\kappa(state)$设定）：
  * 满意则确认 $V-A$（默认 $A=E_c(t)$）；不满则开争议阻断自动结清以谋求调整；
  * 极端（买方即刻没收）：收益差 $\Delta_C=E_c(t)-A\ge0$；$A=E_c(t)$ 打平，$A<E_c(t)$ 则付款严格更优（此阶段 $\kappa=1$）。
* **卖方**：期待 $A-R(t)>0$，不会主动 `forfeit`（为 0）的动机。
* **时间压力**：沉默无利可图——不争议就自动结清。
* **结论**：确认或开争议是理性选择；双方均无自毁动机。

### 3.4 Disputing

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| `acceptPriceBySig(orderId, amountToSeller, proposer, sig)` | 对手方 | 仅 Disputing；验 EIP‑712/1271 签名、nonce、deadline；`amountToSeller ≤ escrow` | → Settled |
| `requestForfeit(orderId,reason)` | Client/Contractor | — | → Forfeited |
| `timeoutForfeit(orderId)` | 任何 | `now ≥ disputeStart + T_dis` | → Forfeited |


* **卖方**：`forfeit`=0，劣于可能的 $A-R(t)>0$，理性上不按键。
* **买方**（见1.1的$\kappa(state)$设定）：
  
  * 直接/到期 `forfeit`→付款与没收的比较由 $E_c(t)$ 与 $A$ 的关系决定（由于 $A\le E_c(t)$，通常付款不劣于没收）。
* **结论**：本阶段仅就"金额"协商；可多轮离线报价，对手方上链接受即一次性结清；谈不拢则由硬截止/没收终止。

**结算与协商语义**：
- 终态 Settled 的收益写作 $(V-A, A-C)$，其中 $A\in[0,E_c(t_{settle})]$ 为协商/确认的实际结清额；
- 未支付部分 $(\text{escrow}-A)$ 原路退回 client；仅支持“单次结清、一次提现”；
- `acceptPriceBySig(...)` 的最小字段：$\{orderId, amountToSeller(=A), proposer, nonce, deadline\}$，并注明“每条报价单用（防重放）”；
- **事件**：成功协商触发 `PriceSettled(orderId, proposer, acceptor, amountToSeller)`，并兼发 `Settled(...)`。

### 3.5 Settled

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| `withdrawPayout(orderId)` | Contractor | — | 承包方提取收益（状态保持） |
| `withdrawRefund(orderId)` | Client | 若有余额 | 客户提取退款（状态保持） |

* **性质**：相较 Forfeited 与 Cancelled，为双方**帕累托更优**。
* **事件**：触发`Settled(...)`事件；若为协商结算则同时触发`PriceSettled(...)`事件。

### 3.6 Forfeited

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| — | — | — | 终态，无进一步动作 |

* **到达**：即时 `requestForfeit` 或 `Disputing` 超时。
* **收益**：$(-E_c(t),0)$。对 $\mathsf{S}$ 劣于任何正利润结清；对 $\mathsf{C}$ 最多与"付款"打平，但可能会留下信誉损失。
* **结论**：**可达但非优**；更多是强制终止的边界。

### 3.7 Cancelled

| 动作 | 发起方 | 守卫/前置条件 | 结果/转移 |
|---|---|---|---|
| — | — | — | 终态，无进一步动作 |

* **到达**： `cancelOrder`，调用条件见上
* **收益**：$(0,0)$。为非对抗性终止。


---


## 5. 均衡（SPE）与参数条件

**结论（充分条件）**
若满足

存在 $A \in (C, \min(V, E_c(t_{settle}))]$ 且 $\kappa(\text{Reviewing})=\kappa(\text{Disputing})=1$。

则按阶段向后归纳：

* 卖方在各子博弈中的极端（即刻没收）在 §4 的边界 $R(t)<A$ 下为**严格劣**；
* 买方在各子博弈中的极端与付款的比较由 $E_c(t)$ 与 $A$ 的关系决定（见 §4），在 $A\le E_c$ 的守卫下付款不劣于没收；
* 因此 **(交付并结清)** 是**子博弈完美均衡**（SPE）的均衡路径；极端动作仅是去中心化边界下的**可达但非优**选择。

**口径切换说明**：未进入争议的阶段默认 $A=E_c(t)$；进入争议后以协商 $A\in[0, E_c(t)]$ 为准，便于沿第3章时序比较。

> 注：在纯收益打平处（买方侧），任意微小的链上摩擦（时间/气费）都会把均衡自然推向"付款/确认"。

---

## 6. 出金逻辑（Pull 结算，A2A 友好）

**总览**
- 入金（escrow/top‑up）使用 push：client 主动转入（ETH: `msg.value`；ERC‑20: `safeTransferFrom`），统一“余额差”记账。
- 出金（结清/退款/没收）使用 pull：状态变更时只“记账可领额”，实际转账由受益方在独立提现函数中完成。

**出金场景**
- Settled（结清）
  - 触发：`approveReceipt`（全额 escrow）、`timeoutSettle`（全额 escrow）、`acceptPriceBySig`（协商额 A）。
  - 记账：
    - 卖方应得 `owedToSeller = amountToSeller`（escrow 或 A）
    - 买方剩余 `refundToBuyer = escrow - amountToSeller`（若 A < escrow）
  - 提现：
    - 卖方 `withdrawPayout(orderId)`（pull）
    - 买方 `withdrawRefund(orderId)`（pull，若有剩余）
- Cancelled（取消）
  - 触发：`Initialized` 任一方；或 `Executing` "从未 markReady 且到期"由 client 取消；或 `Executing`/`Reviewing` 阶段 Contractor 可取消。
  - 记账：`refundToBuyer = escrow`
  - 提现：client 调 `withdrawRefund(orderId)`
- Forfeited（没收）
  - 触发：`requestForfeit` 或 `Disputing` 超时 `timeoutForfeit`。
  - 记账：`escrow → ForfeitPool`；双方应得清零；无对外提现。
- 其它：`Disputing` 仅协商金额；不出金，直到 Settled 或 Forfeited。

**金额计算**
- 全额结清：`amountToSeller = escrow`
- 价格型：`amountToSeller = A`（守卫：$0 \le A \le \text{escrow}$，与第3.4一致）
- 余额返还：`refundToBuyer = escrow - amountToSeller`
- 若使用比例路径（历史兼容）：`amountToSeller = floor(escrow * num / den)`；余数全额返还给 client。

**接口与事件（FST 口径，详细定义见3.4）**
- 结清（仅金额）
  - `acceptPriceBySig(...)`
  - 事件：`PriceSettled(orderId, proposer, acceptor, amountToSeller)`；兼发 `Settled(...)`。
- 提现（pull）
  - 卖方：`withdrawPayout(orderId)`
  - 买方：`withdrawRefund(orderId)`
- 取消与没收
  - 取消：`cancelOrder(orderId)`（见状态守卫）
  - 没收：`requestForfeit(orderId)`、`timeoutForfeit(orderId)`（ForfeitPool 不对外分配）
- 扩展操作
  - 延期：`extendDue(orderId, newDueTs)`、`extendReview(orderId, newRevSec)`（见第3章具体条件）
  - 托管：`topUpEscrow(orderId, amount)`（单调增加，见第1.1节定义）

**安全与实现要点**
- Pull 支付 + `ReentrancyGuard`：提现函数加 `nonReentrant`，遵循 checks‑effects‑interactions。
- `SafeERC20` + 余额差核验：禁止直接 `transfer/transferFrom/approve`；对非常规代币使用白名单策略。
- 入口前抢占：任一外部入口（含提现）先检查是否超时（`now ≥ disputeStart + T_dis`）并处理没收。
- 单次支付不变量：每单最多一次成功卖方提现（`single_payout`）。
- 幂等与标记：提现成功后标记已领，防重复。

**A2A（agent↔agent）建议**
- 事件驱动自动领取：监听 `PriceSettled/Settled`，立刻调用 `withdraw...()`，带重试与回退策略。
- 批量领取：提供 `withdrawMany(orderIds[])` 降低 gas 成本。
- Gasless/代付：EIP‑4337/2771 或元交易（签名领取）以降低运维摩擦。
- Permissionless 领取（可选）：提供任意人代领并给予小额 tip，确保最终性。

---

## 7. 可输出数据（链下）

### 7.1 链下可计算指标与采样窗口
- **争议率/没收率/成功率**：按终态统计订单分布比例
- **平均完成时间**：从Initialized到Settled的时间分布
- **报价接受率**：`acceptPriceBySig`成功率与协商轮次分析
- **$A / E_c^{\text{baseline}}$ 分布**：实际结清额相对托管基线（可选 $E_c^{ready}$ 或 $E_c^{dispute}$）的比例分析
- **Top‑up 轨迹**：$E_c(t)/E_c(0)$ 的放大倍数与轨迹（若分析口径允许）
- **即刻没收/延时动作发生率**：极端策略的实际采用频次

### 7.2 分层分析与版本化口径
- **按阶段分层**：不同状态下的行为模式对比
- **按人群分层**：新用户vs老用户的策略差异
- **按金额分层**：小额vs大额订单的风险偏好
- **采样窗口**：使用最终性窗口（7天确认期）与版本化口径确保数据一致性
