# RFC–PACT/1.0

# PACT — A Two-Party Forfeiture-Backed Escrow Protocol

**副标题：** Trust-Minimized Cooperation for Decentralized Economies  
**作者：** Kirk Brennan, Robert Chen, Maya Patel  
**日期：** 2025-09-26  
**类别：** 标准跟踪  
**版本：** 1.0

---

## 摘要（Abstract）

PACT是一套去中心化的双方托管协议，通过对称威慑机制实现免信任合作。协议专注于资金托管、状态管理和争议解决，不涉及任务内容评价或第三方裁决。核心创新在于相互确保毁灭（MAD）的经济激励设计，确保诚实合作成为占优策略均衡。

### 核心特性
- **成功零分成**：激励双方诚实合作
- **失败对称没收**：通过forfeiture创造威慑均衡
- **开始门控**：startWork() 前任一方可无损取消
- **软硬时钟结合**：T_rev 促进效率，T_dis 强制终止
 - **单边自伤权（SHR）**：参与者可单边执行只削弱自身且不损害对手权利的操作（如延长期限），价值分配仍需双签
- **安全优先设计**：形式化验证的状态机和不变量保护
- **可选元数据支持**：支持少量哈希或URI用于协议扩展

### 安全保证
```
核心不变量：
INV1: 资金守恒 - 托管金总量保持不变
INV2: 状态一致性 - 状态转移满足FSM约束  
INV3: 单次支付 - 每个订单只能有一个受益人
INV4: 时钟正确性 - 计时器行为符合协议规范
```

PACT按"信任最小化（Zero-Trust）"前提运行：正确性来自托管 + 公共超时 + 没收，而非对提交物的链上评价或第三方裁判。

---

## 1. 理论基础：博弈论分析

### 1.1 激励相容性证明

#### 博弈设定
```
参与者：P = {Client, Contractor}
策略空间：S = {Honest, Malicious}  
支付矩阵：
                 Contractor
                 Honest  | Malicious
Client  Honest   (V-E, P)| (-E, -E)
        Malicious (-E, -E)| (-E, -E)

其中：V = 交付价值，P = 支付金额，E = 托管金额
```

#### 纳什均衡分析
**定理1**：当 E ≥ max(P, V-P) 时，(Honest, Honest) 是唯一纳什均衡。

**证明**：
- 对Client：如果Contractor选择Honest，Client选择Honest获得V-E，选择Malicious获得-E，故Honest占优
- 对Contractor：如果Client选择Honest，Contractor选择Honest获得P，选择Malicious获得-E，当P > -E（总是成立）时Honest占优
- 任何涉及Malicious的策略组合都导致双方获得-E，严格劣于(Honest, Honest)

### 1.1.1 时序博弈与子博弈完美均衡（SPNE）

为贴合协议的“阶段式流程（start → signalReady → confirmReceipt → openDispute → acceptSettlement/requestForfeit）”，将双方互动建模为有限时序博弈：

- 阶段顺序：Client 入金与下单 → Contractor 决策是否投入与交付 → Client 评审与是否开启争议 → 双方在 Dispute 中协商或一方触发没收。
- 策略空间：在各阶段上双方可选 Honest（投入/如实评审/按比例和解）或 Deviate（偷懒/无理拒绝/早触发没收）。

子博弈分析要点：
- 在 Dispute 子博弈的末端，任一方“早触发没收”的即期收益均为 −E；而继续协商的保留价值不小于 0（至少可以原路退出到 −E）。因此在合理阈值下，理性方不会在无对价时提前触发没收。
- 沿时序向前归纳，若满足阈值 E ≥ max(P, V−P)，则：
  - Contractor 的偏离（不交付）会引致对方触发没收，其期望 ≤ −E，而按 Honest 合作可得 ≥ P，故不偏离；
  - Client 的偏离（无理拒付/早触发没收）带来 −E，而按 Honest 合作可得 ≥ V−P，故不偏离。

结论（SPNE）：在 E ≥ max(P, V−P) 时，逐阶段向后归纳得到的策略剖面为(Honest, Honest) 在每个子博弈中均为最优反应，构成子博弈完美纳什均衡；由此早触发没收在均衡路径及所有偏离路径上均无利可图。

直觉与阈值解读：
- E ≥ P 使承包方“毁约获利”被彻底压制（因最差落到 −E）；
- E ≥ V−P 使客户“拖延/拒付”被压制（拒付不如合作获得 V−P）；
- 若不确定性较大，可采用更强阈值 E ≥ P ∧ E ≥ V−P 的安全裕度，并在运维侧对异常博弈模式实施监控（见5.3、9.2）。

### 1.2 forfeiture机制的威慑效应

#### 威慑均衡理论
```
传统合约执行：依赖外部执法（法院、仲裁）
PACT机制：依赖相互确保毁灭（Mutual Assured Destruction）

关键洞察：
• 当违约成本 ≥ 违约收益时，理性行为者不会违约
• forfeiture机制确保违约成本固定且显著
• 对称威慑比单方威慑更稳定
```

**定理2**：在 `requestForfeit` 机制下，任何参与者都有能力对对方实施最大惩罚，从而形成威慑均衡。

---

## 2. 安全属性与形式化验证

### 2.1 关键不变量定义

#### 资金安全不变量
```solidity
// INV1: 资金守恒
invariant fund_conservation:
    forall order: Order, time t1, t2:
        total_funds(order, t1) = total_funds(order, t2) OR
        exists valid_transfer: funds_transferred_via(valid_transfer)

// INV2: 单次支付保证  
invariant single_payout:
    forall order: Order:
        count(successful_withdrawals(order)) <= 1

// INV3: 状态机一致性
invariant state_consistency:
    forall order: Order, transition T:
        valid_transition(order.prev_state, T, order.new_state)
```

#### 时态安全属性
```
LIVENESS: 
    □◊(state ∈ {Settled, Forfeited, Cancelled})  // 总是最终终止

SAFETY:
    □(funds_never_disappear ∧ no_double_spending)   // 资金安全

FAIRNESS:
    □(equal_forfeit_access)                         // 对称威慑权利
```

### 2.2 运行时安全检查

#### 强制性安全修饰符
```solidity
// 所有状态变更必须包含的检查
modifier securityChecks(uint256 orderId) {
    uint256 balanceBefore = address(this).balance;
    _;
    
    // 后置条件验证
    require(
        address(this).balance + totalWithdrawn + forfeitPool >= initialBalance,
        "FUNDS_LEAKED"
    );
    
    require(
        orders[orderId].escrowAmount >= 0,
        "INVALID_ESCROW_AMOUNT"
    );
    
    emit SecurityCheckPassed(orderId, block.timestamp);
}
```

#### 紧急安全机制
```solidity
// 经时间锁的紧急暂停机制
contract PACTEmergency {
    uint256 constant EMERGENCY_DELAY = 24 hours;
    uint256 public emergencyProposedAt;
    
    function proposeEmergencyPause() external onlyGovernance {
        emergencyProposedAt = block.timestamp;
        emit EmergencyProposed(block.timestamp);
    }
    
    function executeEmergencyPause() external onlyGovernance {
        require(
            block.timestamp >= emergencyProposedAt + EMERGENCY_DELAY,
            "TIMELOCK_NOT_EXPIRED"
        );
        
        _pause();
        emit EmergencyActivated(block.timestamp);
    }
}
```

---

## 3. 协议状态机设计

### 3.1 状态定义

#### 状态空间（形式化定义）
```
States = {PendingStart, InProgress, UnderReview, Dispute, Settled, Forfeited, Cancelled}

状态不变量：
• PendingStart: escrowAmount > 0 ∧ startTime = 0
• InProgress: startTime > 0 ∧ T_due 有效
• UnderReview: signalTime > 0 ∧ T_rev 计时中
• Dispute: disputeTime > 0 ∧ T_dis 计时中  
• Settled: payoutEnabled = true
• Forfeited: escrowAmount → ForfeitPool
• Cancelled: escrowAmount → Client
```

### 3.2 状态转移表

| State | Trigger | Guard | Effect |
|-------|---------|-------|--------|
| **PendingStart** | `startWork` | `msg.sender = contractor` | → InProgress |
| **PendingStart** | `cancel` | `msg.sender ∈ parties` | → Cancelled |
| **InProgress** | `signalReady` | `msg.sender = contractor ∧ now < startTime + T_due` | → UnderReview |
| **InProgress** | `confirmReceipt` | `msg.sender = client` | → Settled |
| **InProgress** | `cancel` | `msg.sender = client ∧ now ≥ startTime + T_due ∧ neverReady` | → Cancelled |
| **InProgress** | `requestForfeit` | `msg.sender ∈ parties` | → Forfeited |
| **InProgress** | `openDispute` | `msg.sender ∈ parties` | → Dispute |
| **UnderReview** | `confirmReceipt` | `msg.sender = client` | → Settled |
| **UnderReview** | `triggerAutoReceiptConfirm` | `T_rev ≥ T_rev_max` | → Settled |
| **UnderReview** | `requestForfeit` | `msg.sender ∈ parties` | → Forfeited |
| **UnderReview** | `openDispute` | `msg.sender ∈ parties` | → Dispute |
| **Dispute** | `acceptSettlement` | `双方签名` | → Settled |
| **Dispute** | `acceptSettlement` | `双方签名 ∧ 回到工作` | → InProgress |
| **Dispute** | `requestForfeit` | `msg.sender ∈ parties` | → Forfeited |
| **Dispute** | `triggerForfeit` | `T_dis ≥ T_dis_max` | → Forfeited |
| **Settled** | `withdrawPayout` | `msg.sender = contractor` | 资金转移 |
| **Forfeited** | — | — | escrowAmount → ForfeitPool |
| **Cancelled** | — | — | escrowAmount → Client |

**要点**：`signalReady` 只是流程信号，不携带内容、不生成"版本"，仅用于开始/重置 T_rev。协议对"提交物"完全不可见。

### 3.3 状态转移图

#### 形式化状态机定义
```
States: S = {PendingStart, InProgress, UnderReview, Dispute, Settled, Forfeited, Cancelled}
Initial State: s₀ = PendingStart
Final States: F = {Settled, Forfeited, Cancelled}

Transition Function: δ : S × Σ → S
where Σ = {startWork, cancel, signalReady, openDispute, requestForfeit, 
          confirmReceipt, triggerAutoReceiptConfirm, acceptSettlement, 
          triggerForfeit, withdrawPayout}

Transitions:
• δ(PendingStart, startWork) = InProgress
• δ(PendingStart, cancel) = Cancelled
• δ(InProgress, signalReady) = UnderReview
• δ(InProgress, openDispute) = Dispute
• δ(InProgress, requestForfeit) = Forfeited
• δ(InProgress, cancel) = Cancelled    // guard: by client ∧ no Ready ever signaled ∧ now ≥ startTime + T_due
• δ(InProgress, confirmReceipt) = Settled
• δ(UnderReview, confirmReceipt) = Settled
• δ(UnderReview, triggerAutoReceiptConfirm) = Settled
• δ(UnderReview, openDispute) = Dispute
• δ(UnderReview, requestForfeit) = Forfeited
• δ(Dispute, acceptSettlement) ∈ {Settled, InProgress}
• δ(Dispute, requestForfeit) = Forfeited
• δ(Dispute, triggerForfeit) = Forfeited
```

---

## 4. 接口定义

### 4.1 核心接口

#### 创建订单
```solidity
function initByClient(
  address client, 
  address contractor, 
  Token token,
  uint256 escrowAmountDeclared, 
  uint64 T_due, 
  uint64 T_rev, 
  uint64 T_dis,
  bytes32 metadataHash,        // 可选：扩展数据哈希
  string calldata metadataURI   // 可选：扩展数据URI
) external returns (uint256 orderId);
```

#### 工作流程
```solidity
function startWork(uint256 orderId) external;              
function signalReady(uint256 orderId) external;            
/// SHR：仅在 InProgress 中由 client 单边延长到期时间；只能延后（newDueTs > 旧值）
function extendDueByClient(uint256 orderId, uint64 newDueTs) external; 
```
说明：订单创建并入金成功后，订单初始状态为 `PendingStart`。此时承包方（contractor）需要调用 `startWork(orderId)` 以“接受并开工”，状态才会进入 `InProgress`。

#### 确认与争议
```solidity
function confirmReceipt(uint256 orderId) external;                 
function openDispute(uint256 orderId, string calldata reason) external;   
```

#### 自动操作与没收
```solidity
function triggerAutoReceiptConfirm(uint256 orderId) external;      
function triggerForfeit(uint256 orderId) external;         
function requestForfeit(uint256 orderId, string calldata reason) external; 
function cancel(uint256 orderId) external;
function withdrawPayout(uint256 orderId) external;         
```

实现说明（时钟与抢占）：
- `signalReady(orderId)` 仅在 `now < startTime + T_due` 有效；到期后禁止提交 Ready。若需继续，应先延长到期（见 `extendDueByClient`），或进入 Dispute 通过 Settlement 另行约定。
- `extendDueByClient(orderId, newDueTs)`：仅 client 可在 `InProgress` 中调用，且 `newDueTs` 必须晚于当前到期时间；该操作为 SHR（只延长自己的等待窗口），不改变对手方任何权利与资金路径。
- 运行时抢占（MUST）：实现应在所有对外入口前检查定时器：
  - 若 `state == Dispute ∧ now ≥ disputeStart + T_dis`，应当立即切换为 `Forfeited`（对称没收），再终止本次调用；`triggerForfeit` 可保留为显式“催更”入口但非必需。
  - 若 `state == UnderReview ∧ !inDispute ∧ now ≥ readyAt + T_rev`，应当自动 `Settled`，再终止本次调用；以区分自动与手动的路径（事件见 8. 事件日志）。

实现说明：
- 当 `triggerAutoReceiptConfirm(orderId)` 成功使状态转入 `Settled` 时，合约实现应当（MUST）发出 `AutoReceiptConfirmed(orderId)` 事件，以区分自动确认与手动 `confirmReceipt` 的路径，便于链下 KPI 计算。
- `cancel(orderId)` 的允许条件：
  - 在 `PendingStart`：任一方可取消，escrow 全额返还 client；
  - 在 `InProgress`：仅当 `从未 signalReady` 且 `当前时间 ≥ startTime + T_due` 且 `未处于 Dispute` 时，client 可取消并全额返还；
  - 其他状态禁止取消。

### 4.2 资金操作（Top-up，最小扩展）

```solidity
/// 仅 client 可调用；仅增加、不减少。
function topUpEscrowETH(uint256 orderId) external payable;

/// ERC-20 版本；金额以余额差（credited）入账。
function topUpEscrow(uint256 orderId, uint256 amount) external;
```

约束与语义：
- 允许状态：`PendingStart | InProgress | UnderReview | Dispute`；禁止：`Settled | Forfeited | Cancelled`。
- 仅允许“单调增加”，`amount > 0`，并遵守最小/最大托管额限制。
- 采用余额差记账：`credited = balanceAfter - balanceBefore`；`orders[orderId].escrowAmount += credited`。
- 不改变任何时钟或状态；需 `nonReentrant` 与资金守恒后置校验。

### 4.3 状态查询接口

```solidity
interface IPACTStateQuery {
    struct OrderStatus {
        uint256 orderId;
        State currentState;
        uint256 timeInCurrentState;
        uint256 timeToNextDeadline;
        bool canTriggerAutoReceiptConfirm;
        bool canRequestForfeit;
        bytes32 metadataHash;
        string metadataURI;
    }
    
    function getOrderStatus(uint256 orderId) 
        external view returns (OrderStatus memory);
    
    function getTimeouts(uint256 orderId) 
        external view returns (uint256 T_due, uint256 T_rev, uint256 T_dis);
}
```

### 4.4 元数据管理

```solidity
struct Order {
    // ... 核心字段
    bytes32 metadataHash;    // 可选：指向外部数据的哈希
    string metadataURI;      // 可选：外部数据URI
}

function setOrderMetadata(
    uint256 orderId,
    bytes32 dataHash,
    string calldata uri
) external authorizedParty(orderId) {
    orders[orderId].metadataHash = dataHash;
    orders[orderId].metadataURI = uri;
    emit MetadataUpdated(orderId, dataHash, uri);
}
```

### 4.5 订单意向（可选扩展，Off‑FSM）

为支持由承包方发起的“提案/意向”，在不改变主状态机的前提下提供下列可选接口：

```solidity
function createIntentByContractor(
    address client,
    Token token,
    uint256 escrowAmountDeclared,
    uint64 T_due,
    uint64 T_rev,
    uint64 T_dis,
    bytes32 metadataHash,
    string calldata metadataURI
) external returns (uint256 intentId);

function cancelIntent(uint256 intentId) external;

// 由 client 入金并落地为正式订单，初始状态为 PendingStart
function initFromIntent(uint256 intentId) external payable returns (uint256 orderId);
```

语义：意向对象不进入 FSM；只有当 client 通过 `initFromIntent` 成功入金后，才创建正式订单并进入 `PendingStart`（满足 `escrowAmount > 0` 的状态不变量）。随后承包方需调用 `startWork(orderId)` 以接受并进入 `InProgress`。

### 4.6 额外补偿（建议做法，协议外）

如需在和解时对承包方进行额外补偿，推荐采用“协议外直接转账”的方式完成（ETH 或 ERC‑20），并通过 `setOrderMetadata(orderId, metadataHash, metadataURI)` 记录补偿交易哈希或说明，以便审计检索。该额外补偿不并入 `escrow`，亦不计入 Forfeit 基线。

---

## 5. 安全实现要求

### 5.1 强制性安全模式

#### 核心安全要求
```solidity
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol"; // 统一使用 SafeERC20
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract PACTv1_0 is ReentrancyGuard, Pausable {
    using SafeMath for uint256;
    
    // 安全常量
    uint256 constant MAX_URI_LENGTH = 2048;           // URI最大长度（示例实现，可调整或移除）
    
    // 强化的修饰符组合（示例不再统一发出 StateTransition 事件）
    modifier safeStateTransition(uint256 orderId, State expectedState) {
        require(orders[orderId].state == expectedState, "INVALID_STATE");
        require(!paused(), "CONTRACT_PAUSED");
        _;
    }
    
    modifier authorizedParty(uint256 orderId) {
        require(
            msg.sender == orders[orderId].client || 
            msg.sender == orders[orderId].contractor,
            "UNAUTHORIZED"
        );
        _;
    }
    
    modifier fundConservation(uint256 orderId) {
        uint256 balanceBefore = address(this).balance;
        _;
        require(
            address(this).balance + totalWithdrawn + totalForfeited == 
            initialTotalBalance,
            "FUND_CONSERVATION_VIOLATED"
        );
    }
    
    // 注意：合约内不对托管金额做上下限强制校验；任何额度策略建议在产品/风控层实现。
}
```

#### 元数据验证
```solidity
// 强制元数据验证
function validateMetadata(bytes32 hash, string calldata uri) internal pure {
    if (hash != bytes32(0)) {
        require(hash != bytes32(0), "INVALID_HASH");
    }
    if (bytes(uri).length > 0) {
        require(bytes(uri).length <= MAX_URI_LENGTH, "URI_TOO_LONG");
        // 基础URI格式检查
        require(bytes(uri)[0] != 0x00, "INVALID_URI_FORMAT");
    }
}

// 增强的订单创建
function initByClient(
    address client,
    address contractor,
    Token token,
    uint256 escrowAmountDeclared,
    uint64 T_due,
    uint64 T_rev,
    uint64 T_dis,
    bytes32 metadataHash,
    string calldata metadataURI
) external 
    nonReentrant
    returns (uint256 orderId) {
    
    validateMetadata(metadataHash, metadataURI);
    
    // 时间参数验证
    require(T_due >= 3600, "T_due_TOO_SHORT");        // 至少1小时
    require(T_due <= 30 days, "T_due_TOO_LONG");      // 最多30天
    require(T_rev >= 1800, "T_rev_TOO_SHORT");        // 至少30分钟
    require(T_rev <= 7 days, "T_rev_TOO_LONG");       // 最多7天
    require(T_dis >= 24 hours, "T_dis_TOO_SHORT");    // 至少24小时
    require(T_dis <= 30 days, "T_dis_TOO_LONG");      // 最多30天
    
    // 继续创建订单逻辑...
}
```

### 5.2 经济参数计算与建议值（链下）

本节为运营侧“链下建议”。默认产品策略可采用“E = P（价格即托管额）”；合约不强制、不校验 E/P 比例；任何计算器与默认档位请参考“附录 B：运营建议与示例（链下）”。

### 5.3 行为监控与量化指标（链下）

声明：KPI 仅在链下通过事件计算；核心规范不内置任何链上统计。核心三项 KPI 见 9.2；更多算法示例参见“附录 A（KPI‑EXT）”与“附录 B（运营建议与示例）”。

#### 合谋检测算法
```typescript
// 量化的合谋检测参数
const COLLUSION_THRESHOLDS = {
    neverDisputeThreshold: 20,        // 连续20次无争议视为可疑
    abnormalEscrowRatio: 0.1,         // 托管金额低于正常值10%
    timingWindowSeconds: 300,         // 5分钟内的协调操作
    minSampleSize: 10,                // 最少10笔交易才进行检测
    highRiskScore: 0.8,               // 高风险阈值
    mediumRiskScore: 0.5              // 中等风险阈值
};

interface CollusionMetrics {
    neverDisputeCount: number;
    abnormalEscrowRatio: number;
    coordinatedActionCount: number;
    totalTransactions: number;
    riskScore: number;
}

function detectCollusionPatterns(
    participantA: address,
    participantB: address,
    orders: Order[]
): CollusionRisk {
    const pairOrders = orders.filter(o => 
        (o.client === participantA && o.contractor === participantB) ||
        (o.client === participantB && o.contractor === participantA)
    );
    
    if (pairOrders.length < COLLUSION_THRESHOLDS.minSampleSize) {
        return { riskLevel: 0, status: "INSUFFICIENT_DATA" };
    }
    
    const metrics: CollusionMetrics = {
        neverDisputeCount: pairOrders.filter(o => o.disputes.length === 0).length,
        abnormalEscrowRatio: calculateAbnormalEscrowRatio(pairOrders),
        coordinatedActionCount: detectCoordinatedActions(pairOrders),
        totalTransactions: pairOrders.length,
        riskScore: 0
    };
    
    // 计算综合风险分数
    metrics.riskScore = calculateCollusionRiskScore(metrics);
    
    return {
        riskLevel: metrics.riskScore,
        metrics: metrics,
        recommendedAction: getRecommendedAction(metrics.riskScore)
    };
}

function calculateCollusionRiskScore(metrics: CollusionMetrics): number {
    let score = 0;
    
    // 从不争议模式 (权重: 0.4)
    const neverDisputeRatio = metrics.neverDisputeCount / metrics.totalTransactions;
    if (neverDisputeRatio > 0.95) score += 0.4;
    else if (neverDisputeRatio > 0.85) score += 0.2;
    
    // 异常托管模式 (权重: 0.3)
    if (metrics.abnormalEscrowRatio > 0.3) score += 0.3;
    else if (metrics.abnormalEscrowRatio > 0.15) score += 0.15;
    
    // 协调行为模式 (权重: 0.3)
    const coordinationRatio = metrics.coordinatedActionCount / metrics.totalTransactions;
    if (coordinationRatio > 0.5) score += 0.3;
    else if (coordinationRatio > 0.3) score += 0.15;
    
    return Math.min(score, 1.0);
}

function getRecommendedAction(riskScore: number): string {
    if (riskScore >= COLLUSION_THRESHOLDS.highRiskScore) {
        return "IMMEDIATE_INVESTIGATION";
    } else if (riskScore >= COLLUSION_THRESHOLDS.mediumRiskScore) {
        return "ENHANCED_MONITORING";
    } else {
        return "ROUTINE_MONITORING";
    }
}
```

#### 系统健康监控指标
```typescript
interface SystemHealthMetrics {
    // 成功率指标
    overallSuccessRate: number;        // 总体成功率 (目标: >95%)
    disputeRate: number;               // 争议率 (目标: <5%)
    forfeitureRate: number;            // 没收率 (目标: <2%)
    
    // 效率指标  
    averageCompletionTime: number;     // 平均完成时间
    autoReceiptConfirmRate: number;    // 自动确认率 (目标: <10%)
    revisionRate: number;              // 修改要求率 (目标: <20%)
    
    // 经济指标
    averageEscrowRatio: number;        // 平均托管比例
    totalValueLocked: number;          // 总锁定价值
    economicEfficiency: number;        // 经济效率 (成功交易价值/总锁定价值)
}

const HEALTH_THRESHOLDS = {
    minSuccessRate: 0.95,              // 最低成功率
    maxDisputeRate: 0.05,              // 最高争议率  
    maxForfeitureRate: 0.02,           // 最高没收率
    maxAutoReceiptConfirmRate: 0.10,   // 最高自动确认率
    maxRevisionRate: 0.20,             // 最高修改率
    minEconomicEfficiency: 0.85        // 最低经济效率
};

function assessSystemHealth(orders: Order[]): SystemHealthStatus {
    const metrics = calculateHealthMetrics(orders);
    const issues: string[] = [];
    
    if (metrics.overallSuccessRate < HEALTH_THRESHOLDS.minSuccessRate) {
        issues.push("LOW_SUCCESS_RATE");
    }
    if (metrics.disputeRate > HEALTH_THRESHOLDS.maxDisputeRate) {
        issues.push("HIGH_DISPUTE_RATE");
    }
    if (metrics.forfeitureRate > HEALTH_THRESHOLDS.maxForfeitureRate) {
        issues.push("HIGH_FORFEITURE_RATE");
    }
    
    return {
        status: issues.length === 0 ? "HEALTHY" : "NEEDS_ATTENTION",
        metrics: metrics,
        issues: issues,
        recommendedActions: generateHealthRecommendations(issues)
    };
}
```

---

## 6. 争议与和解

### 6.1 争议处理机制
- **仅在 Dispute 阶段允许协商**
- **协商项**：部分结算（payoutRatioNum/Den，向下取整；余数归 Client）、范围变更/延期（回 InProgress）、终止（立即结清）
- **双签生效**；T_dis 超时未一致 → Forfeited（双方为 0，Pool 得款）
- **任一方可直接申请没收**：`requestForfeit()` 立即进入 Forfeited，无需等待超时

说明（支付语义）：
- 协议不支持“多次分期支付”；“部分结算”指按比例一次性结清，随后仅允许一次 `withdrawPayout`。
- 如需在接受和解时表达 client 的加价让步，建议在协议外直接向承包方地址转账（ETH/代币），并可使用 `setOrderMetadata` 记录 `bonusTxHash` 等信息以留痕；该额外支付不并入 `escrow`、不计入 Forfeit。

示例：`escrow=100`，约定 `payoutRatio=3/4`，则承包方可提取 `floor(100*3/4)=75`，余数 `25` 返还给 client`。若客户另行补偿 `10`，则可直接转账并在元数据记录交易哈希。

### 6.2 Settlement接口
```solidity
struct SettlementTerms {
    uint256 payoutRatioNum;
    uint256 payoutRatioDen;
    uint64 newDueTime;        // 如果延期
    bool returnToProgress;    // 是否回到InProgress
    bool immediateTermination; // 立即终止
    uint256 nonce;            // 重放保护（每order或每对手方递增）
    uint64  deadline;         // 截止时间（区块时间戳）
}

function proposeSettlement(uint256 orderId, SettlementTerms calldata terms) external;
function acceptSettlement(uint256 orderId, bytes32 termsHash) external;
```

#### 6.2.1 PriceOffer（默认路径，价格唯一定义）

为收敛“Dispute 仅讨论金额”，推荐采用价格型报价 + 单函数接受：

```solidity
struct PriceOffer {
    uint256 orderId;
    uint256 amountToSeller;   // 绝对金额
    uint256 nonce;            // per-order per-proposer 递增
    uint64  deadline;         // 截止时间（区块时间戳）
}

function acceptPriceBySig(
    uint256 orderId,
    PriceOffer calldata offer,
    address proposer,
    bytes calldata sig
) external;
```

说明与守卫（MUST）：
- 仅在 `Dispute`/`Disputing` 状态；调用方必须为对手方；
- 验证 `offer` 的 EIP‑712 签名（或合约钱包 ERC‑1271）；`block.timestamp <= deadline`；
- 防重放：每提案方在每单上维护 `nonce` 递增；接受成功后推进最小 nonce；
- 金额边界：`0 ≤ amountToSeller ≤ currentEscrow(orderId)`；一次性结清，余数返还 client；
- 入口前抢占：若 `now ≥ disputeStart + T_dis`，应先进入 `Forfeited` 再拒绝本次调用；
- 事件（建议）：`PriceOffered(orderId, proposer, amount)`（可选）；`PriceSettled(orderId, proposer, acceptor, amount)`（MUST）。

#### 签名与验证（EIP‑712 + ERC‑1271，建议/可选）

建议采用 EIP‑712 结构化签名与 ERC‑1271 校验，以便离线签署与多钱包兼容；但这不是协议内核的强制要求。核心语义可通过链上 `proposeSettlement/acceptSettlement` 双调用完成。

- 推荐支持两类签章主体：
- 外部账户（EOA）：使用 ECDSA 恢复验证；
- 合约钱包（ERC‑1271）：使用 `isValidSignature` 验证。

EIP‑712 域：
```solidity
string constant NAME = "PACT";                 // 或 "PACT-Settlement"
string constant VERSION = "1";
// 域字段：name, version, chainId, verifyingContract
```

Typed‑Data 定义（含 orderId 参与摘要）：
```solidity
bytes32 constant SETTLEMENT_TYPEHASH = keccak256(
    "SettlementTerms(uint256 orderId,uint256 payoutRatioNum,uint256 payoutRatioDen,uint64 newDueTime,bool returnToProgress,bool immediateTermination,uint256 nonce,uint64 deadline)"
);

function _hashSettlement(uint256 orderId, SettlementTerms memory t) internal pure returns (bytes32) {
    return keccak256(abi.encode(
        SETTLEMENT_TYPEHASH,
        orderId,
        t.payoutRatioNum,
        t.payoutRatioDen,
        t.newDueTime,
        t.returnToProgress,
        t.immediateTermination,
        t.nonce,
        t.deadline
    ));
}

function _termsDigest(bytes32 domainSeparator, uint256 orderId, SettlementTerms memory t) internal pure returns (bytes32) {
    return keccak256(abi.encodePacked("\x19\x01", domainSeparator, _hashSettlement(orderId, t)));
}
```

验签流程（建议实现）：
- `proposeSettlement(orderId, terms)`: 计算 `termsHash = _hashSettlement(orderId, terms)`，记录并发出 `SettlementProposed(orderId, termsHash, ...)`；记录 `terms.nonce` 与 `terms.deadline`；可选地在此阶段校验提议者签名并持久化。
- `acceptSettlement(orderId, termsHash)`: 从存储中取回 `terms`，计算 `digest = _termsDigest(domainSeparator, orderId, terms)`，并对“对手方”的签名进行下列之一验证：
  - EOA：`recover(digest, sig) == expectedCounterparty`；
  - ERC‑1271：`IERC1271(expectedCounterparty).isValidSignature(digest, sig) == 0x1626ba7e`。
- 额外校验（MUST）：`block.timestamp <= terms.deadline`；`terms.nonce == expectedNonce(orderId, counterparty)`；验签成功后递增 `expectedNonce`，并将 `used[orderId][termsHash] = true`（或依赖 nonce 即可，二者取其一）。

兼容性说明：本节只定义摘要与验证规范，不强制更改 6.2 的函数签名；实现可通过事件与存储把 `termsHash → terms` 进行绑定，并在 `acceptSettlement` 时完成双签闭环。

---

## 7. 资金与记账

### 7.1 资金流向
- **Settled**：向 Contractor 开放 `withdrawPayout`；平台不抽成
- **Forfeited**：escrowAmount 须全额进入 ForfeitPool
- **Cancelled**：全额返还 Client（允许情形：PendingStart 任何一方；InProgress 且从未 signalReady 且已达 `startTime + T_due` 由 client 取消）
- **余额差记账（强制）**：`credited = balanceAfter - balanceBefore`；`escrowAmount = credited`
  - Top-up（最小扩展）：按余额差增量入账，更新 `escrowAmount += credited`；不改变任何时钟或状态；支持 `Dispute` 阶段 Top-up。
  - 协议外补偿（推荐）：客户可在和解时或和解后对承包方进行直接转账补偿；该金额不并入 `escrow`、不计入 Forfeit，可通过元数据记录交易哈希用于审计。
  - 实现要求：
    - 所有 ERC‑20 交互必须使用 `SafeERC20`：`safeTransfer` / `safeTransferFrom` / `safeIncreaseAllowance` / `safeDecreaseAllowance`；禁止直接调用 `transfer` / `transferFrom` / `approve`。
    - 如需重置授权，先将额度降为 0，再增加到目标值，避免已知的授权竞态。
    - 对返回值不可靠/非标准的代币，以余额差核验为准。
    - 可选：支持 EIP‑2612 `permit` 以减少授权交互次数。

### 7.2 支持的资产
- **默认仅支持常规 ERC‑20 与 ETH**（实现必须使用 `SafeERC20` 包装所有调用）
- **非常规代币**（fee‑on‑transfer、rebasing、ERC‑777 hooks）默认禁用
- **白名单启用**：仅在审计与风控通过后启用，且必须配合余额差核验；必要时做“等值核验”（如滑点/手续费导致的入账差额需在应用侧明确约定）。

---

## 8. 事件日志

```solidity
event OrderCreated(uint256 orderId, address client, address contractor, Token token,
                   uint256 escrowAmount, bytes32 metadataHash, string metadataURI);
event SellerStarted(uint256 orderId);                             
event ReadySignaled(uint256 orderId, address by);                 
event ReceiptConfirmed(uint256 orderId);                                  
event AutoReceiptConfirmed(uint256 orderId);                       // 与 ReceiptConfirmed 区分自动通过
event DisputeOpened(uint256 orderId, address by);
event DueExtendedByClient(uint256 orderId, uint64 newDueTs);
event SettlementProposed(uint256 orderId, bytes32 termsHash,
                         uint256 payoutRatioNum, uint256 payoutRatioDen,
                         uint64 newDue, bool termination);
event SettlementAccepted(uint256 orderId, bytes32 termsHash);
event Settled(uint256 orderId, uint256 amountToSeller,
              uint256 payoutRatioNum, uint256 payoutRatioDen, uint256 buyerRemainder);
// 价格型结算（默认路径）
event PriceOffered(uint256 orderId, address proposer, uint256 amountToSeller);
event PriceSettled(uint256 orderId, address proposer, address acceptor, uint256 amountToSeller);
event ForfeitRequested(uint256 orderId, address by, string reason);
event Forfeited(uint256 orderId);
event OrderCancelled(uint256 orderId, uint8 phase);
event EscrowToppedUp(uint256 orderId, address by, uint256 amount, uint256 newEscrow);
event IntentCreated(uint256 intentId, address contractor, address client, Token token,
                    uint256 escrowAmountDeclared, bytes32 metadataHash, string metadataURI);
event IntentCancelled(uint256 intentId, address by);
event MetadataUpdated(uint256 orderId, bytes32 dataHash, string uri);
```

**说明**：事件支持元数据记录，允许协议扩展层存储必要的上下文信息。为保持事件粒度清晰且便于索引，本规范不新增统一的 `StateTransition` 事件；如需统一态迁移事件，建议另行作为可选扩展提案评审。

---

## 9. 部署与监控要求

### 9.1 部署前检查清单

#### 安全审计要求
- [ ] **智能合约审计**：至少2家独立审计公司完成审计
- [ ] **形式化验证**：关键不变量的数学证明完成
- [ ] **渗透测试**：模拟攻击测试通过
- [ ] **代码覆盖率**：测试覆盖率达到95%以上
- [ ] **Gas优化验证**：主要操作gas消耗在合理范围内

#### 经济模型验证
- [ ] **博弈论证明**：激励相容性数学证明审查
- [ ] **参数压力测试**：极端参数下的系统行为验证
- [ ] **经济攻击模拟**：合谋攻击等场景测试
- [ ] **流动性分析**：不同市场条件下的资金效率

#### 运营准备
- [ ] **监控系统**：实时监控dashboard部署
- [ ] **报警机制**：异常情况自动报警设置
- [ ] **紧急响应**：安全事件响应团队和流程
- [ ] **用户文档**：完整的API文档和使用指南
- [ ] **合规审查**：法律和监管要求确认

### 9.2 持续监控要求

#### 实时安全监控
- **不变量检查**：每个区块验证INV1-INV4
- **异常交易检测**：gas使用量、失败率等异常监控
- **重入攻击防护**：ReentrancyGuard状态实时监控
- **时间锁验证**：紧急暂停机制的完整性检查

#### 经济行为分析
- **成功率监控**：目标>95%，低于90%触发警报
- **争议率监控**：目标<5%，超过10%需要调查
- **合谋检测**：使用5.3节的量化算法
- **托管效率**：总锁定价值vs实际使用效率

#### 系统健康指标
- **TPS监控**：交易处理能力和延迟
- **错误率跟踪**：失败交易的分类统计
- **用户活跃度**：日活用户和交易量趋势
- **经济指标**：平均订单价值、完成时间等

#### 报警阈值设置
```typescript
const ALERT_THRESHOLDS = {
    criticalAlerts: {
        successRate: 0.90,           // 成功率低于90%
        disputeRate: 0.10,           // 争议率超过10%
        systemDowntime: 300,         // 系统停机超过5分钟
        securityBreach: true         // 任何安全漏洞
    },
    warningAlerts: {
        successRate: 0.95,           // 成功率低于95%
        disputeRate: 0.05,           // 争议率超过5%
        highGasUsage: 1000000,       // 单操作gas超过100万
        collusionRisk: 0.5           // 合谋风险超过0.5
    }
};
```

---

#### KPI 计算与公开化（链下）
- 开源子图/ETL：应提供开源的 subgraph 或 ETL 脚本以复现 KPI 指标计算。
- 计算口径版本化：为所有指标的口径/算法建立版本号与变更记录，确保可追溯。
- 最终性窗口：在 KPI 统计与告警中应用“最终性窗口”（finality window），以规避重组/跨层延迟导致的数据抖动。

#### 运维护栏（非协议约束）
- Top‑up 运营策略：设定“日内加押增幅上限（如 ≤1.5×）”“账户级日加押次数上限（如 ≤5 次）”，触发阈值时仅做前端限制与告警，不改变合约 FSM。
- 审计留痕：对于超常 Top‑up 或异常 E/P 比订单，要求在 `metadataURI` 留存说明或外部凭证指针，便于后审。
- 只读监控：上述限制由前后端与监控系统执行，链上合约不内置强制规则，确保协议内核极简。

---

## 10. 协议边界与扩展性

### 10.1 协议范围

#### ✅ PACT协议负责：
- 双方托管资金管理
- 订单状态生命周期
- 争议解决和没收机制
- 安全和经济保证
- 元数据哈希和URI存储

#### ❌ PACT协议不负责：
- 任务内容描述和质量评估
- 参与者发现和匹配
- 具体业务逻辑实现
- 复杂的链上争议裁决

### 10.2 扩展机制

#### 元数据扩展
协议通过可选的元数据字段支持扩展：
```solidity
struct Order {
    // ... 核心协议字段
    bytes32 metadataHash;    // 可选：扩展数据哈希
    string metadataURI;      // 可选：外部数据URI
}
```

#### 建议的扩展格式
```json
{
  "version": "1.0",
  "protocol_extension": "task_specification",
  "data": {
    "task_type": "string",
    "requirements": {...},
    "acceptance_criteria": {...}
  }
}
```

#### 扩展边界
- ✅ **协议负责**：元数据存储和验证
- ❌ **协议不解析**：元数据具体内容
- 🔧 **应用层负责**：元数据格式定义和解析

### 10.3 品牌与术语使用指南

- 正式名称：协议名称为“PACT”，推荐副标题为“对称没收托管协议”（英文：Forfeiture‑Backed Escrow）。
- 关于“去中心化支付宝”表述：
  - 仅作为类比用于科普与解释“先托管、后放款”的用户心智，不作为正式名称、主标语或法务文件用语。
  - 禁止将“支付宝”或其他注册商标用于产品名、域名、Logo、应用商店简介、首页主Slogan、合约/法律文件命名，避免商标混同与误导。
  - 类比使用时须附上免责声明，说明双方无关联且 PACT 不提供中心化仲裁/KYC/退款保障，仅提供链上托管、状态机与超时/没收机制。
- 更稳妥的替代表述（建议）：
  - “去中心化托管支付” / “可信中立的双边托管协议” / “对称威慑的零信任托管” / “协议层托管与结算”。
- 免责声明示例（中文）：
  - “本文将 PACT 类比为‘去中心化的支付宝’，仅用于帮助理解托管流程。PACT 与支付宝/蚂蚁集团无任何关联；PACT 不提供中心化仲裁、KYC 或退款保障，仅提供链上托管、状态机与超时/没收机制。”

---

## 11. 合规清单

- ✅ **协议支持少量元数据存储**：metadataHash 和 metadataURI 用于扩展
- ✅ **仅维护状态与时钟**；`signalReady()` 仅为流程信号
- ✅ **`startWork()` 前取消** → Cancelled（全额返还）
- ✅ **InProgress 未 signalReady 且 T_due 到期**：client 可 `cancel()` → Cancelled（全额返还）
- ✅ **UnderReview 才计 T_rev**；每次 `signalReady()` 重置
- ✅ **非 Dispute 且 T_rev 到期应自动结清（`AutoReceiptConfirmed`）**；在 Dispute 禁止
- ✅ **任一方可 `openDispute()`**；任一方可 `requestForfeit()` 立即没收
- ✅ **Dispute 到 `T_dis` 应立即没收（对称威慑），可选保留 `triggerForfeit` 入口**
- ✅ **Settlement 双签**；部分结算向下取整；余数归 Client
- ✅ **不支持多次支付**；仅支持“按比例一次性结清 + 单次提现”
- ✅ **资金流**：Settled→`withdrawPayout`；Forfeited→ForfeitPool；Cancelled→Client
- ✅ **余额差记账**；非常规代币需白名单 + 安全措施
- ✅ **ReentrancyGuard + pull-payment**；公共超时入口开放
- ✅ **（核心）InProgress 到期后禁止 `signalReady`；client 可用 `extendDueByClient` 单边延后到期（SHR）**
- ✅ （扩展）Top-up：仅 `client` 可调；允许状态为 `PendingStart | InProgress | UnderReview | Dispute`；仅单调增加
- ✅ （建议）协议外补偿：客户可直接转账补偿并用元数据记录 `bonusTxHash`，不并入 `escrow`、不计入 Forfeit
- ✅ （扩展）订单意向：承包方可创建意向，正式订单仍由 client 入金后方可生效

- ✅ 对外术语与商标：不使用“支付宝”等商标词作为正式名称/标语；如作类比，须附免责声明；对外建议使用“去中心化托管支付”等中性表述。

---

## 附录 A：KPI‑EXT 指标（链下示例）

说明：以下指标为“可选扩展”，用于社区二次开发与运营分析，均在链下依据事件计算，不属于协议内核。计算应采用最终性窗口，并为口径配置版本号。

- E/P 比分布 ep_ratio_percentiles_[window]
  - 定义：对每笔已结清订单计算 `E/P = escrowAmount / amountToSeller`；在给定窗口内输出分位数（P50/P90/P99）。
  - 事件来源：`OrderCreated`（escrowAmount）、`Settled`（amountToSeller）。
  - 告警建议（可选）：E/P > 2.5 或 E/P < 0.8 视为离群，需人工复核业务合理性。

- Top‑up 活跃度 topup_activity_[window]
  - 定义：账户级/订单级的 Top‑up 次数与幅度（如日内最大增幅倍数）。
  - 事件来源：`EscrowToppedUp`。
  - 告警建议（可选）：单订单日内加押倍数 > 1.5× 或 账户日加押次数 > 5 触发黄/红牌。

- AutoReceiptConfirm→Forfeit 漏斗率 autoreceiptconfirm_to_forfeit_[window]
  - 定义：在窗口内 `AutoReceiptConfirmed` 后仍进入 `Forfeited` 的订单比例。
  - 事件来源：`AutoReceiptConfirmed`、`Forfeited`。
  - 告警建议（可选）：该比例理论上≈0；≥1% 需排查竞态/实现缺陷或异常用法。

— 以上为参考口径，社区可通过 kpi‑ext 清单（JSON/YAML）发布自研指标与公式。

---

## 附录 B：运营建议与示例（链下）

以下示例均为链下参考，不构成协议内核要求：

### B.1 托管金额计算示例
```typescript
// 推荐的托管金比例表（示例口径）
const ESCROW_RATIOS = {
  lowRisk: 1.5,
  mediumRisk: 2.0,
  highRisk: 3.0,
  critical: 5.0
};

// 推荐的最小托管金额（USD 口径，仅示例）
const MIN_ESCROW_USD = { micro: 10, small: 100, medium: 1000, large: 10000 };

function calculateOptimalEscrow(
  paymentAmount: bigint,
  taskCategory: 'micro'|'small'|'medium'|'large',
  riskLevel: 'lowRisk'|'mediumRisk'|'highRisk'|'critical'
): bigint {
  const ratio = ESCROW_RATIOS[riskLevel];
  const calculatedEscrow = paymentAmount * BigInt(Math.ceil(ratio));
  const minEscrowWei = usdToWei(MIN_ESCROW_USD[taskCategory]);
  return calculatedEscrow > minEscrowWei ? calculatedEscrow : minEscrowWei;
}
```

### B.2 时间参数推荐示例
```typescript
const TIME_CONFIGS = {
  T_due: { urgent: 3600, normal: 86400, complex: 604800, research: 2592000 },
  T_rev: { simple: 1800, normal: 7200, detailed: 86400, thorough: 259200 },
  T_dis: { standard: 172800, extended: 604800, complex: 1209600 }
};

function calculateTimeouts(
  taskType: 'urgent'|'normal'|'complex'|'research',
  reviewDepth: 'simple'|'normal'|'detailed'|'thorough',
  disputeComplexity: 'standard'|'extended'|'complex'
): {T_due:number,T_rev:number,T_dis:number} {
  return {
    T_due: TIME_CONFIGS.T_due[taskType],
    T_rev: TIME_CONFIGS.T_rev[reviewDepth],
    T_dis: TIME_CONFIGS.T_dis[disputeComplexity]
  };
}
```

### B.3 合谋检测伪代码（链下）
```typescript
const COLLUSION_THRESHOLDS = {
  neverDisputeThreshold: 20,
  abnormalEscrowRatio: 0.1,
  timingWindowSeconds: 300,
  minSampleSize: 10,
  highRiskScore: 0.8,
  mediumRiskScore: 0.5
};
// 其余算法细节同先前示例，可根据业务自行调整。
```

— 上述内容为运营团队的参考模板，产品可按需取舍或替换。

---

## 12. 结语

PACT协议通过对称威慑机制解决了去中心化环境下的双方合作问题。协议专注于托管核心功能，避免了复杂的主观判断和第三方依赖，为Web3协作经济提供了可证明安全的基础设施。

通过明确的协议边界设计，PACT为上层应用和扩展机制留出了充分的创新空间，同时保持了协议本身的简洁性和可审计性。元数据支持机制为协议扩展提供了标准化的接口，使PACT能够适应不同应用场景的需求。

---

**作者：**
- Kirk Brennan - 主要作者，博弈论分析和协议设计
- Robert Chen - 安全审查和形式化验证
- Maya Patel - 协议扩展性和应用场景分析

**协议定位：** 去中心化双方合作的托管基础设施

**End of RFC–PACT/1.0**
