# ERC‑PACT：双方托管与争议结算最小接口（草案）

eip: TBA
title: ERC‑PACT — Minimal Two‑Party Forfeiture‑Backed Escrow
author: <your_name> <your@email>
discussions-to: TBA
status: Draft
type: Standards Track
category: ERC
created: 2025-09-26
requires: 165
license: CC0-1.0

---

## 摘要（Abstract）

ERC‑PACT 定义了一个“链上支付宝”式的最小托管接口：链上仅维护资金托管、状态机与时钟，不做任务内容评价或链上聚合统计。协议支持一次性结清（成功零分成）、对称没收（失败威慑）、公共超时，且明确禁止“多次分期支付”。规范使用 ERC‑165 进行接口发现，并将 Top‑up/Typed‑Data/双方发起等能力拆分为可选扩展。

## 动机（Motivation）

- 为AI agent/服务/订单类应用提供统一、可审计、可组合的托管与争议处理接口，降低集成成本。
- 将链上职责收敛到“资金 + 状态 + 时钟”，把 KPI、经济参数、身份/治理等留给链下或上层扩展，避免过载共识。
- 保持事件与状态机可重放、可索引，便于 L2/钱包/索引器/审计工具生态协同。

## 非目标（Out of Scope）

- 经济参数计算器与默认档位（链下运营）；
- KPI/监控聚合（链下，通过事件实现）；
- 仲裁/身份/公共品资助/跨层治理等上层机制；
- 复杂代币适配（fee‑on‑transfer、rebasing、ERC‑777 hooks 默认不支持）。

## 术语与状态机

- 参与方：client（付款方/买方）、contractor（收款方/卖方）。
- 资产：`address token`（`address(0)` 表示原生 ETH）。
- 时钟：`T_rev`（评审，仅 UnderReview）、`T_dis`（争议，仅 Dispute）。
- 状态集合：`PendingStart, InProgress, UnderReview, Dispute, Settled, Forfeited, Cancelled`。

状态与时钟约束（MUST）：
- 初始 `PendingStart`：client 入金创建；contractor 必须 `startWork` 接单后进入 `InProgress`。
- `InProgress → UnderReview`：contractor `signalReady`；仅在 `now < startTime + T_due` 允许；`signalReady` SHOULD 重置 `T_rev`。
- `InProgress` 逾期未就绪（MUST）：若从未 `signalReady` 且当前时间 ≥ `startTime + T_due`，client 有权 `cancel()` 并全额返还；其他情形禁止取消。
- `UnderReview`：允许 `confirmReceipt/openDispute/requestForfeit`；非 Dispute 且 `T_rev` 到期自动 `Settled`（可保留 `triggerAutoReceiptConfirm` 作为显式入口）。
- `Dispute`：允许 `propose/acceptSettlement`、`requestForfeit`；到 `T_dis` 应立即 `Forfeited`（可保留 `triggerForfeit` 作为显式入口）。
- 终态：`Settled`（contractor 单次 `withdrawPayout`）、`Forfeited`（全额入 ForfeitPool）、`Cancelled`（仅 `PendingStart`，全额返 client）。
- MUST：`T_rev` 仅在 `UnderReview` 计时；在 `Dispute` 禁止 `triggerAutoReceiptConfirm`。
- MUST：`T_dis` 仅在 `Dispute` 计时；实现应在入口前抢占到期并 `Forfeited`。

## 结算语义（单次、按比例）

结算项（MUST）：
- 结构体 `SettlementTerms { payoutRatioNum, payoutRatioDen, newDueTime, returnToProgress, immediateTermination, nonce, deadline }`；
- 计算：`amountToSeller = floor(escrowAmount × payoutRatioNum / payoutRatioDen)`；余数 MUST 返还 client；
- 约束：`0 ≤ payoutRatioNum ≤ payoutRatioDen` 且 `payoutRatioDen > 0`；
- 单次支付不变量：每个订单仅允许一次成功提现；不支持分期多次支付；
- MAY：`returnToProgress=true` 表示部分结清后回到 `InProgress` 继续工作。

## 规范（Specification）

接口分层使用 ERC‑165 发现。实现 MUST 返回对应 interfaceId。

### Core（MUST）

```solidity
pragma solidity ^0.8.20;

interface IERC_PACT_Core /* is IERC165 */ {
    enum State { PendingStart, InProgress, UnderReview, Dispute, Settled, Forfeited, Cancelled }
    struct Timeouts { uint64 T_due; uint64 T_rev; uint64 T_dis; }

    // 双方可发起：
    // - 若 initiator = Client，则本调用应附带入金（ETH: msg.value；ERC20: 见实现）并直接创建 PendingStart；
    // - 若 initiator = Contractor，则创建“待入金”订单，需 client 后续 fundByClient 才有效。
    enum Initiator { Client, Contractor }

    // 建议同时提供便捷版 initByClient（MAY），但 MUST 提供 initByEither + fundByClient 以满足“双方可发起”的需求。
    function initByEither(
        Initiator initiator,
        address otherParty,
        address token,
        uint256 escrowAmount,
        Timeouts calldata t,
        bytes32 metadataHash,
        string calldata metadataURI
    ) external payable returns (uint256 orderId);

    // 若由 contractor 发起，client 需补入金以使订单进入 PendingStart（仍需 contractor startWork 接单）
    function fundByClient(uint256 orderId, uint256 amount) external payable;

    // 便捷版：client 直接入金创建（MAY）
    function initByClient(
        address contractor,
        address token,
        uint256 escrowAmount,
        Timeouts calldata t,
        bytes32 metadataHash,
        string calldata metadataURI
    ) external payable returns (uint256 orderId);

    // 承包方接单（Required）：PendingStart → InProgress
    function startWork(uint256 orderId) external;

    // 流程推进
    function signalReady(uint256 orderId) external;                // contractor
    function confirmReceipt(uint256 orderId) external;             // client; InProgress/UnderReview → Settled
    function openDispute(uint256 orderId, string calldata reason) external;   // either
    function triggerAutoReceiptConfirm(uint256 orderId) external;  // any, not in Dispute
    function requestForfeit(uint256 orderId, string calldata reason) external; // either
    function triggerForfeit(uint256 orderId) external;             // any, Dispute && T_dis expired
    function cancel(uint256 orderId) external;                     // PendingStart: either; InProgress: client-only if neverReady && due expired

    // SHR：Client 单边延长 T_due（仅 InProgress；只能延后）
    function extendDueByClient(uint256 orderId, uint64 newDueTs) external;

    // 结算提案与接受（比例一次性结清）
    struct SettlementTerms {
        uint256 payoutRatioNum;
        uint256 payoutRatioDen;
        uint64  newDueTime;         // optional
        bool    returnToProgress;   // optional
        bool    immediateTermination;
        uint256 nonce;              // replay protection
        uint64  deadline;           // unix timestamp expiry
    }
    function proposeSettlement(uint256 orderId, SettlementTerms calldata terms) external; // either
    function acceptSettlement(uint256 orderId, bytes32 termsHash) external;               // counterparty

    // 提现（单次）
    function withdrawPayout(uint256 orderId) external;             // contractor, only Settled
}
```

### Query（MUST）

```solidity
pragma solidity ^0.8.20;

interface IERC_PACT_Query /* is IERC165 */ {
    struct OrderStatus {
        uint256 orderId;
        uint8   currentState;           // cast of State
        uint256 timeInCurrentState;
        uint256 timeToNextDeadline;
        bool    canTriggerAutoReceiptConfirm;
        bool    canRequestForfeit;
        bytes32 metadataHash;
        string  metadataURI;
    }
    function getOrderStatus(uint256 orderId) external view returns (OrderStatus memory);
    function getTimeouts(uint256 orderId) external view returns (uint256 T_due, uint256 T_rev, uint256 T_dis);
}
```

### Settlement（MUST）

事件与语义见“事件”章节；实现 MUST 按“比例一次性结清”计算应付金额并确保余数返还。

### 可选扩展（MAY）

Top‑up（最小加押，client‑only；包含 Dispute，终态禁止）：
```solidity
pragma solidity ^0.8.20;
interface IERC_PACT_TopUp /* is IERC165 */ {
    function topUpEscrowETH(uint256 orderId) external payable;             // client-only
    function topUpEscrow(uint256 orderId, uint256 amount) external;        // client-only, ERC20
}
```

价格型结算（PriceOffer，默认路径，可选扩展接口）：
```solidity
pragma solidity ^0.8.20;
interface IERC_PACT_SettlementPrice /* is IERC165 */ {
    struct PriceOffer {
        uint256 orderId;
        uint256 amountToSeller;   // 绝对金额，结清给承包方
        uint256 nonce;            // per-order per-proposer 递增
        uint64  deadline;         // 截止时间（区块时间戳）
    }

    // 仅在 Disputing；acceptor 必须是对手方；校验 EIP-712 签名
    function acceptPriceBySig(
        uint256 orderId,
        PriceOffer calldata offer,
        address proposer,
        bytes calldata sig
    ) external;
}
```

评审期延期（SHR，可选）：
```solidity
pragma solidity ^0.8.20;
interface IERC_PACT_ReviewExtension /* is IERC165 */ {
    // Contractor 单边延长评审期，仅在 UnderReview；只能延长
    function extendReviewBySeller(uint256 orderId, uint64 newTrevSec) external;
    event ReviewExtendedBySeller(uint256 indexed orderId, uint64 newTrevSec);
}
```

Intent（Off‑FSM，若未采用 `initByEither` 路线）：
```solidity
pragma solidity ^0.8.20;
interface IERC_PACT_Intent /* is IERC165 */ {
    function createIntentByContractor(
        address client,
        address token,
        uint256 escrowAmount,
        IERC_PACT_Core.Timeouts calldata t,
        bytes32 metadataHash,
        string calldata metadataURI
    ) external returns (uint256 intentId);
    function cancelIntent(uint256 intentId) external;
    function initFromIntent(uint256 intentId) external payable returns (uint256 orderId);
}
```

Typed‑Data Settlement（EIP‑712 + ERC‑1271，建议/可选）：
```solidity
pragma solidity ^0.8.20;
interface IERC_PACT_SettlementTypedData /* is IERC165 */ {
    // SettlementTerms 已含 nonce/deadline；此扩展仅定义签名打包格式。
    struct SignedSettlementTerms {
        IERC_PACT_Core.SettlementTerms terms;
        bytes   signature; // EOA: ECDSA；合约钱包：ERC-1271
    }
}
```

## 事件（Events，MUST）

实现 MUST 发出如下事件（字段尽量 indexed 以利索引）。

```solidity
event OrderCreated(uint256 indexed orderId, address indexed client, address indexed contractor,
                   address token, uint256 escrowAmount, bytes32 metadataHash, string metadataURI);
event SellerStarted(uint256 indexed orderId);
event ReadySignaled(uint256 indexed orderId, address by);
event ReceiptConfirmed(uint256 indexed orderId);
event AutoReceiptConfirmed(uint256 indexed orderId);
event DisputeOpened(uint256 indexed orderId, address by);
event DueExtendedByClient(uint256 indexed orderId, uint64 newDueTs);
event SettlementProposed(uint256 indexed orderId, bytes32 termsHash,
                         uint256 payoutRatioNum, uint256 payoutRatioDen,
                         uint64 newDue, bool termination);
event SettlementAccepted(uint256 indexed orderId, bytes32 termsHash);
event Settled(uint256 indexed orderId, uint256 amountToSeller,
              uint256 payoutRatioNum, uint256 payoutRatioDen, uint256 buyerRemainder);
// 价格型结算事件（默认路径，建议至少发 PriceSettled）
event PriceOffered(uint256 indexed orderId, address indexed proposer, uint256 amountToSeller);
event PriceSettled(uint256 indexed orderId, address indexed proposer, address indexed acceptor, uint256 amountToSeller);
event ForfeitRequested(uint256 indexed orderId, address by, string reason);
event Forfeited(uint256 indexed orderId);
event OrderCancelled(uint256 indexed orderId, uint8 phase);
event MetadataUpdated(uint256 indexed orderId, bytes32 dataHash, string uri);

// 扩展事件（TopUp/Intent）
event EscrowToppedUp(uint256 indexed orderId, address by, uint256 amount, uint256 newEscrow);
event IntentCreated(uint256 indexed intentId, address indexed contractor, address indexed client,
                    address token, uint256 escrowAmountDeclared, bytes32 metadataHash, string metadataURI);
event IntentCancelled(uint256 indexed intentId, address by);
```

说明：规范不提供统一 `StateTransition` 事件，保持细粒度事件稳定；若需统一态迁移事件，建议另行扩展。

## 资产与记账（MUST）

- 原生 ETH：以 `msg.value` 入金；
- ERC‑20：使用余额差（balance‑delta）核验入金与结算；
- MUST：所有 ERC‑20 交互使用 `SafeERC20`（`safeTransfer` / `safeTransferFrom` / `safeIncreaseAllowance` / `safeDecreaseAllowance`）；禁止直接 `transfer/approve`；必要时“先清 0 再增额”；
- SHOULD：对非常规代币启用白名单与余额差核验；默认拒绝。

## 不变量与安全（MUST）

- 资金守恒：合约余额 + 已提现 + 已没收 == 初始总额；
- 单次支付：每个订单仅允许一次成功提现；
- 时钟正确性：`T_rev/T_dis` 的使用范围与时序遵循状态机约束；
- 防重入与拉式支付：建议 `ReentrancyGuard` + pull‑payment；
- 访问控制：状态检查、参与方检查（client/contractor），公共入口（自动触发类）限制清晰；
- 错误码建议：`InvalidState`、`Unauthorized`、`NotAllowedDuringDispute`、`InvalidRatio`、`InvalidAmount`、`InvalidToken`。

## 兼容性与部署

- 接口发现：实现 MUST 在 `supportsInterface(bytes4)` 返回 Core/Query/Settlement 以及所支持扩展的 interfaceId；
- L2 优先部署：规范与共识层无关；可在“非规范”章节说明与 ERC‑4337 的协作点（MAY）。

## 参考实现（指南）

- Solidity 0.8.x；严格使用 `SafeERC20` 与余额差核验；
- 拒绝非常规代币或以白名单+余额差启用；
- 事件全覆盖；`AutoReceiptConfirmed` 在自动确认时 MUST 发出；
- 运行时抢占（MUST）：在所有对外入口前先检查并处理到期超时——`Dispute` 过期即 `Forfeited`；`UnderReview` 非争议过期即 `Settled`；
- 结算采用单次比例计算，向下取整；余数返还 client；
- `startWork` 必须由 contractor 调用承接订单（满足“client 付款 + contractor 接单”）。

## 测试向量（MUST）

- 正常流：init → startWork → signalReady → confirmReceipt → withdrawPayout；
- 部分结清（一次性）：比例、取整、余数返还；
- 争议：openDispute → propose/acceptSettlement；requestForfeit；triggerForfeit（超时路径）；
- 取消：`PendingStart` cancel 全额返还；
- Top‑up（扩展）：允许/禁止状态矩阵；
- 资金守恒与单次提现断言；
- 时钟边界：`T_rev`/`T_dis` 到期条件与禁止条件；
- ERC‑20 兼容：非标准返回值代币、授权重置、余额差核验；
- 重入场景：提现与结算的 reentrancy 保护。

## Rationale（设计原理简述）

- 简洁性：一次性结清 + 对称没收 + 公共超时，保证可证明性与可审计性；
- 事件优先：所有 KPI/监控留给链下，规范只定义稳定事件；
- 双方可发起但权责清晰：资金确权在 client 入金；contractor 必须 `startWork` 接单；
- 争议期冻结加押：避免移动威慑基线与“加押/接受/没收”竞态。

## 接口 ID（示例）

实现可在部署时计算并常量化 interfaceId：

```solidity
bytes4 constant IID_IERC_PACT_Core        = type(IERC_PACT_Core).interfaceId;
bytes4 constant IID_IERC_PACT_Query       = type(IERC_PACT_Query).interfaceId;
bytes4 constant IID_IERC_PACT_Settlement  = 0xTBD; // 若 Settlement 独立接口文件，按实际签名计算
bytes4 constant IID_IERC_PACT_TopUp       = 0xTBD;
bytes4 constant IID_IERC_PACT_Intent      = 0xTBD;
bytes4 constant IID_IERC_PACT_TypedData   = 0xTBD;
```

（注意：若将 Core/Settlement 合并为单接口文件，按合并后的签名计算 interfaceId。）

## 安全考虑（Security Considerations）

- 重入与拉式支付：提现/结算路径必须 `nonReentrant`；
- 前跑与竞态：在 Dispute 禁止 Top‑up；自动通过/没收入口需清晰时序与条件；
- 代币风险：对 fee‑on‑transfer/rebasing/777 默认禁用或白名单；仅用余额差核验；
- 只读重放：事件审计与链下 KPI 采用最终性窗口，避免重组抖动；
- 访问控制：严格校验参与方与状态；公共函数（trigger 类）避免 DoS 窗口。

## 版权与许可（License）

本规范采用 CC0‑1.0 许可（公共领域贡献）。
