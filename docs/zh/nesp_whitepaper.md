# A2A 无仲裁托管结算协议（NESP）白皮书
副题：信任最小化 · 限时争议 · 对称没收威慑 · 零手续费

## 版本说明
- 本白皮书为独立、可判定、可复现的完整文档；正文内自洽给出全部定义、判据与口径，不依赖外部引用。
- 本白皮书即为唯一规范性文档（唯一语义源），包含完整的规则、接口与指标口径。

## 0. 摘要（Abstract）
就像互联网的演化一样，唯有“自由创建（无审计）、自由发布（无预审）、自由交易（无许可）”，生态才会自发涌现并持续扩张。然而完全自由也容易滑向“劣币驱逐良币”的黑暗森林。为此，必须有一套能奖善惩恶的 A2A（Agent‑to‑Agent）结算底座，其核心特性是“无仲裁，且能自协作促进”。

- 无仲裁：不设中心化第三方裁判，不做价值判断与平台裁量；合约仅依据可证事实（签名、金额与可验证触发器）执行，保持可信中立与零协议费。
 - 自协作促进：把“不可达一致”的后果预先承诺为对称没收，使拖延与敲诈失去收益空间，将理性最优策略推向限时妥协与及时结清（规范性触发器为“限时到期”；其他触发器仅作信息性可验证信号，不进入守卫）。

NESP 正是这样的底座：链下协商，链上约束；以对称没收威慑为核心，通过统一的时间/金额口径与开放事件实现可复现审计，在不引入裁量的前提下，把自由交易引导回可预期、可合作的轨道，且合约维度保持零协议费（仅网络 Gas 成本）。

### 0.1 核心流程（快速导览，非规范）
- Step 1 托管（createAndDeposit/depositEscrow）：买方将应付资金 E 托管至合约，E 单调不减；授权/来源遵循 2771/4337 记录（via 字段）。口径锚点：SIA3、2.6、3.2、6.3、INV.10。
- Step 2 交付（acceptOrder→markReady）：卖方承接订单进入 Executing（E1），并在 `startTime + D_due` 前宣告就绪（E3），设置 `readyAt` 并起算评审窗。口径锚点：E1/E3、G.E1/G.E3、INV.11/INV.12。
- Step 3 无争议结清（approveReceipt/timeoutSettle）：买方验收（E4/E8）或评审超时（E9）触发全额结清，按 `amountToSeller = E` 记账，并聚合到可提余额（Pull）。口径锚点：E4/E8/E9、INV.1/INV.4/INV.5/INV.10、事件 `Settled/BalanceCredited`。
- Step 4 争议（raiseDispute）：在 Executing/Reviewing 任意时刻进入 Disputing（E5/E10），设置 `disputeStart` 并冻结 E（禁止充值）。口径锚点：E5/E10、G.E5/G.E10、SIA3（ErrFrozen）。
- Step 5 签名协商（settleWithSigs）：争议期内双方就 `A`（A≤E）达成 EIP‑712/1271 可验证签名并由对手方上链结清（E12），差额返还买方。口径锚点：E12、5.1（签名域/nonce/deadline）、INV.2/INV.3。
- Step 6 对称没收（timeoutForfeit）：若 `now ≥ disputeStart + D_dis` 仍未达成一致，触发没收（E13），`escrow → ForfeitPool`（不对外分配，非协议费）。口径锚点：E13、G.E13、INV.8、INV.14。
说明：本小节为导览，规范性口径以第 3–6 章（状态机/不变量/安全/接口与事件）为准；本节中的“口径锚点”仅为阅读帮助，不构成规范条文。

## 1. 设计原则（Principles）

### 1.1 最小内置（Minimal Enshrinement）
- 约束：结算内核不承载裁量与价值判断，不引入仲裁/表决/再质押依赖；链上仅保存可验证的最小集合：状态（订单流转）、金额口径（A≤E，E 单调不减）、可被外界承认的触发信号，以及公开可验证的时钟/窗口。
- 边界：上层的身份/信誉/拍卖/任务分配/使命型机制在应用层实现；账户抽象/可信转发（AA/2771/4337）仅为调用通道，不改变金额/时间/事件口径。
- 禁止项：把“谁对谁错”的裁量写入合约、从结清/托管中扣取任何协议费、以治理投票决定结算结果等。

### 1.2 可信中立（Credible Neutrality）
- 约束：确定性时间窗、对称规则、开放事件；任意第三方可重放审计。
- 证据：公开统一金额/时间口径与最小事件字段，保证“别人不必相信我们，但可以检验我们”。

### 1.3 零协议费（Zero‑Fee）
- 约束：结算合约不从托管 E 或结清 A 中抽取任何协议内费用；仅存在网络 Gas 成本。
- 证据：零费违规计数应=0；如检测到扣费或内置费项，应视为违规并可回退。
- 披露：外部费（钱包/代付/打包/桥/路由价差等）作为信息披露而非协议口径。

### 1.4 可验证与可重放（Verifiable & Replayable）
- 最小证据集：
  - 签名承诺：采用结构化签名，域至少包含 {orderId, tokenAddr, amount, chainId, nonce, deadline}，防跨单/跨链/重放；合约/域错/过期均为统一回滚路径。
  - 金额与会计：E 单调不减、A≤E；
  - 触发器族：定义“不可达一致”的可验证触发信号（限时只是其一；亦可包含签名缺席/矛盾、最低可验证性失败、握手破裂等）；
  - 时钟与窗口：统一时间口径与到期判定，任何人可据此重放路径与结果。

### 1.5 与 A2A 生命周期的对接（Lifecycle Alignment）
- 原则：结算适配不改变 A2A 的消息语义；提供清晰的“消息→结算动作”映射与“结算事件→会话回填”路径，使会话/线程与订单上下文同源一致。
- 调用路径：直连与代付/转发均需记录来源（via 字段），以便审计与归因；调用通道不改变结算口径。

### 1.6 分阶段开放与门槛治理（Phased Opening）
- 门槛：以统一参数表 {W, θ, β, τ} 设定阶段验收与运行门槛（窗口、没收率上限、协商接受率下限、P95 结清上限），并版本化管理。
- 动作：额度/清算/缓冲/熔断等运营动作位于应用层执行，内核不变。
- 目标：随着渗透提升，保持可判定/可复现/可对照与可回退，避免规模外溢把系统推回裁量与黑箱。

## 2. 模型与记号（A ≤ E 口径）

### 2.1 参与者与信息结构
- 参与者：Client（买方）、Contractor（卖方）。
- 公开信息：状态、时间戳、托管额 E、事件与日志；
- 私人信息：V（买方价值）、C（卖方成本）、质量主观信号。

### 2.2 时间与计时器
- 绝对锚点：`startTime, readyAt, disputeStart`（一次性设置后不可回拨）；
- 相对窗口：`D_due, D_rev, D_dis`（单位秒；`D_due/D_rev` 允许“单调延后”，`D_dis` 固定且不可延长）。

### 2.3 金额与单位
- `E`（托管额）：单调不减、不可减少，仅经 `depositEscrow` 增加；
- `A`（实际结清额）：0 ≤ A ≤ E（以结清时刻的 E 计）；
- 单位与时间：金额以 token 最小单位计价；时间以 `block.timestamp`（秒）。

### 2.4 术语与符号（信息性）
- `V`：买方价值；`C`：卖方成本；
- `κ(t)`：可用性/状态系数（Executing∈[0,1]；Reviewing/Disputing=1）；
- `R(t)`/`I(t)`：卖方/买方的机会收益/沉没成本项。

### 2.5 资产与代币
- `tokenAddr` 表示资产标识（ERC‑20 或“原生 ETH”哨兵）；
- ETH 资产：充值入口为 `payable`，MUST 满足 `msg.value == amount`；提现使用 `call` 且 `nonReentrant`；
- ERC‑20：`msg.value == 0`，使用 SafeERC20 `transferFrom` 成功后记账；
- 建议：支持“WETH 适配层”作为工程选项，但规范层必须支持原生 ETH。

### 2.6 参数协商与范围（规范）
- 协商主体与生效时点：`E`、`D_due`、`D_rev`、`D_dis` 由 Client 与 Contractor 针对“每一笔订单”达成一致；实现必须在订单建立/接受时固化存储。
- 默认值：若 `dueSec/revSec/disSec` 传入 0，则采用协议默认 `D_due=1d=86_400s`、`D_rev=1d=86_400s`、`D_dis=7d=604_800s`；入库与事件需记录替换后的“生效值”。
- 修改规则：`E` 仅可单调增加；`D_due/D_rev` 仅允许在争议发生前单调延后；`D_dis` 自设置后固定，不提供延长入口。
- 有界性：三者必须为有限值且大于 0；为抵御重组，`D_dis ≥ 2·T_reorg`（由部署方按目标链给出估计）。
- 零值约定：仅在创建入口可传入 0 表示“采用默认值”，其余接口与持久化字段不得为 0。

### 2.7 “不可达一致”触发器（规范与信息性）
- 规范性触发器（唯一）：限时到期。WHAT：当 `state=Disputing` 且 `now ≥ disputeStart + D_dis` 时，允许 `timeoutForfeit` 将状态置为 `Forfeited`。
- 信息性/可选触发器（非规范）：签名缺席/矛盾、最低可验证性失败、握手破裂等，可用于产品层诊断或外部治理，不进入合约守卫；其动作与罚则不在本规范内定义。
  强提醒：上述信息性触发器仅用于分析/诊断，不进入任何合约守卫或状态转换判据。

## 3. 状态机与守卫

#### 核验要点（WHY/HOW/WHAT）
- WHY：给出唯一允许的状态转移与守卫，形成可审计的有限步终止路径。
- HOW（≤3）：
  1) 依据事件 `Accepted/DisputeRaised/Settled/Forfeited/Cancelled` 重放状态序列；
  2) 用 `startTime/readyAt/disputeStart` 与 `D_due/D_rev/D_dis` 验证 G.E* 守卫；
  3) 在 Disputing 状态调用充值预期 `ErrFrozen`，验证冻结语义。
- WHAT：仅允许 E1–E13 转移；任一其他转移或违反守卫即不通过。

### 3.1 允许的转换（除此之外无其他）
- E1 Initialized -acceptOrder-> Executing（发起：contractor）
- E2 Initialized -cancelOrder-> Cancelled（发起：client/contractor）
- E3 Executing -markReady-> Reviewing（发起：contractor）
- E4 Executing -approveReceipt-> Settled（发起：client）
- E5 Executing -raiseDispute-> Disputing（发起：client/contractor）
- E6 Executing -cancelOrder-> Cancelled（发起：client）
- E7 Executing -cancelOrder-> Cancelled（发起：contractor）
- E8 Reviewing -approveReceipt-> Settled（发起：client）
- E9 Reviewing -timeoutSettle-> Settled（发起：任意）
- E10 Reviewing -raiseDispute-> Disputing（发起：client/contractor）
- E11 Reviewing -cancelOrder-> Cancelled（发起：contractor）
- E12 Disputing -settleWithSigs-> Settled（发起：对手方（client 或 contractor 之一））
- E13 Disputing -timeoutForfeit-> Forfeited（发起：任意）

### 3.2 状态不变动作（SIA）
- SIA1：`extendDue(orderId, newDueSec)`（仅 client）要求 `newDueSec > 当前 D_due`（严格延后）。
- SIA2：`extendReview(orderId, newRevSec)`（仅 contractor）要求 `newRevSec > 当前 D_rev`（严格延后）。
- SIA3：`depositEscrow(orderId, amount)`（`payable`）要求 `amount > 0`，且 `escrow ← escrow + amount`。调用方可以为任意地址：
  - 若调用方为受信路径（2771/4337 等），实现 MUST 解析业务主体 `subject` 并确保 `subject == client`；失败 MUST `revert`（`ErrUnauthorized`）；
  - 其他任意地址视为无条件赠与且自担没收风险，不改变订单权利义务，同时需自行完成代币转移（见下）。
- 冻结/终态/金额守卫顺序：`state ∈ {Disputing}` → `ErrFrozen`；`state ∈ {Settled, Forfeited, Cancelled}` → `ErrInvalidState`；其余进入金额/资产校验。
- 资产口径：订单资产为 ETH 时需 `msg.value == amount`；为 ERC‑20 时需 `msg.value == 0`，令 `payer ≡ subject`，并在 `SafeERC20.transferFrom(payer, this, amount)` 成功后记账（受信路径下 payer=client；赠与路径下 payer 为调用主体，需要其提前授权）。
- 适用范围：SIA3 允许于 Initialized/Executing/Reviewing；在 Disputing 与任何终态禁止充值。

### 3.3 守卫与副作用
- 参数持久化：在订单建立/接受时固化 `D_due/D_rev/D_dis`；不得依赖“隐式默认”。
- 锚点一次性：`startTime/readyAt/disputeStart` 仅允许“未设置→设置一次”。
- 调用主体解析（Resolved Subject）：直连 `subject = msg.sender`（`via = address(0)`）；如调用来自受信转发器（2771），则 `subject = _msgSender()`；如来自 EntryPoint（4337），则 `subject = userOp.sender`。
- 主体约束（MUST）：
  - `markReady`、`extendReview`：`subject == contractor`；
  - `approveReceipt`、`extendDue`：`subject == client`；
  - `raiseDispute`、`settleWithSigs`：`subject ∈ {client, contractor}`；
  - `cancelOrder`：按守卫分支检查 `subject == client`（G.E6）或 `subject == contractor`（G.E7/G.E11）；
  - `timeoutSettle`、`timeoutForfeit`：主体不限（任意地址可触发）。
- G.E1：`acceptOrder` 仅当 `state=Initialized`；发起主体（解析 2771/4337 后的业务主体）MUST 等于该订单的 `contractor`，否则 `ErrUnauthorized`；副作用：`startTime = now`。
- G.E3：`markReady` 仅当 `now < startTime + D_due`；副作用：`readyAt = now` 并起算 `D_rev`。
- G.E4/E8：`approveReceipt` 仅适用于 `state ∈ {Executing, Reviewing}`。
- G.E9：`timeoutSettle` 仅当 `state=Reviewing` 且 `now ≥ readyAt + D_rev`。
- G.E5/E10：`raiseDispute` 允许于 Executing/Reviewing 任意时刻进入 Disputing；副作用：`disputeStart = now`；进入 Disputing 后 E 冻结，任何充值 `revert`（ErrFrozen）。
- G.E11：`cancelOrder`（contractor）仅当 `state=Reviewing`。
- G.E13：`timeoutForfeit` 仅当 `state=Disputing` 且 `now ≥ disputeStart + D_dis`。
- G.E6：`cancelOrder`（client）仅当“从未 Ready（`readyAt` 未设置）且 `now ≥ startTime + D_due`”。
- G.E7：`cancelOrder`（contractor）允许（无额外守卫）。
- G.E12：`settleWithSigs` 仅当 `state=Disputing`，且 `amountToSeller ≤ E` 并通过 EIP‑712/1271 签名、nonce、deadline 校验。

### 3.4 终态约束
- `Settled/Forfeited/Cancelled` 为终态；到达终态后不得再改变状态或资金记账；仅允许提现入口读取并领取既有可领额（若有）。

## 4. 结算与不变量（Pull 语义）

#### 核验要点（WHY/HOW/WHAT）
- WHY：确保 A≤E、单次入账、Pull 提现与零协议费恒等式成立。
- HOW（≤3）：
  1) 对 `Settled/AmountSettled/BalanceCredited/BalanceWithdrawn` 事件做会计对账；
  2) 检查 `escrow_before = amountToSeller + refundToBuyer`（或没收等式）；
  3) 在状态变更入口无外部 `transfer`，提现路径 `nonReentrant`。
- WHAT：INV.1–INV.14 均满足；任一恒等式破坏即不通过。

### 4.1 金额计算
- INV.1 全额结清：`amountToSeller = escrow`（approve/timeout）。
- INV.2 金额型结清：`amountToSeller = A` 且 `0 ≤ A ≤ escrow`（签名协商）。
- INV.3 退款：`refundToBuyer = escrow − amountToSeller`（若 A < escrow）。

### 4.2 资金安全
- INV.4 单次入账：每单至多一次将结清额/退款额入账至聚合余额（single_credit），防止重复计入可提余额。
- INV.5 幂等提现：提现前先读取并清零聚合余额，转账成功即完成；重复调用无可提余额，返回无副作用。
- INV.6 入口前抢占：外部入口先处理 `timeout*`，防延迟攻击。
  - 审计判据：当入口被调用时，若超时条件已满足（例如 `now ≥ readyAt + D_rev` 或 `now ≥ disputeStart + D_dis`），应优先导致对应的 `timeoutSettle/timeoutForfeit` 结果或返回超期错误；若未发生优先处理，视为违反本不变量。
- INV.7 资产与对账：SafeERC20 + 余额差核验；对“费率/重基/非标准”代币如无法保证恒等对账，MUST `revert`（ErrAssetUnsupported）。

### 4.3 资金去向与兼容
- INV.8 没收去向：`escrow → ForfeitPool`（不向外部分配）；ForfeitPool 为合约内逻辑账户，罚没资产留存合约中，不向任何外部地址（含零地址/黑洞）转移；ETH 与 ERC‑20 采用一致语义。
- INV.9 比例路径兼容（可选）：`amountToSeller = floor(escrow * num / den)`；余数全部计入买方退款。实现需使用安全“mulDiv 向下取整”或等效无溢出实现；任何溢出/下溢必须回滚；禁止四舍五入与提精度。
- INV.10 Pull 语义：状态变更仅“记账可领额”（聚合到 `balance[token][addr]`），实际转账仅在 `withdraw(token)` 发生；禁止在状态变更入口直接 `transfer`。
- INV.11 锚点一次性：`startTime/readyAt/disputeStart` 一旦设置，MUST NOT 修改或回拨。
- INV.12 计时器规则：`D_due/D_rev` 仅允许延后（单调增加，且在进入 Disputing 前）；`D_dis` 固定且不可延长。
- INV.13 唯一机制：无争议路径必须全额结清；争议路径采用签名金额结清；金额口径始终满足 `A ≤ E`，链上仅记录托管与结清。
- INV.14 零协议费恒等式：任一结清/没收动作发生时，满足 `escrow_before = amountToSeller + refundToBuyer` 或（没收）`escrow_before = forfeited`；不允许出现协议费扣减。违反恒等式必须 `revert`（ErrFeeForbidden）。

#### 审计提示（信息性）
- HOW（≤3）：
  1) 对没收路径：按 `tokenAddr` 分组比对“合约资产余额增量 = Σ(Forfeited.amount)”且随后无 `BalanceCredited/BalanceWithdrawn` 外流；
  2) 核对订单维度的 `owed/refund` 清零；
  3) 对 ETH 与 ERC‑20 采用一致口径（余额差 + 事件）。
- WHAT：罚没资产留存合约，不向外部分配或销毁。

## 5. 安全与威胁模型

#### 核验要点（WHY/HOW/WHAT）
- WHY：防重放、域错/过期校验、授权与重入安全。
- HOW（≤3）：
  1) 构造跨单/跨链/过期签名验证 `ErrBadSig/ErrExpired/ErrReplay`；
  2) 2771/4337 授权失败验证 `ErrUnauthorized` 且不发 `EscrowDeposited`；
  3) 提现复用 `nonReentrant` 并遵循 CEI 的路径检查。
- WHAT：错误映射完整、授权与回滚路径一致。

### 5.1 签名与重放
- 采用 EIP‑712/1271；签名域至少包含 `{chainId, contract, orderId, tokenAddr, amountToSeller(=A), proposer, acceptor, nonce, deadline}`。
- `amountToSeller ≤ E`；`nonce` 的作用域至少为 `{orderId, signer}` 且一次性消费；`deadline` 基于 `block.timestamp` 判定。
- 必须防止跨订单/跨合约/跨链重放。

### 5.2 错误映射（统一）
- `ErrInvalidState`、`ErrGuardFailed`、`ErrAlreadyPaid`、`ErrExpired`、`ErrBadSig`、`ErrOverEscrow`、`ErrFrozen`、`ErrFeeForbidden`、`ErrAssetUnsupported`、`ErrReplay`、`ErrUnauthorized`。

### 5.3 交互顺序与重入
- 提现 `nonReentrant`，遵循 CEI；优先处理 `timeout*` 入口；提现使用 `call` 并检查返回。
- CEI 范围（规范）：除提现与 ERC‑20 `transferFrom` 外，所有状态变更入口禁止直接向外部地址转账或调用外部不受信任合约；所有状态变更入口均遵循“先校验、再记账、后交互”的顺序。

### 5.4 时间边界与残余风险
- 统一区块时间；`D_due/D_rev` 仅允许延后（单调），`D_dis` 固定且不可延长。
- 残余风险：破坏型对手导致的没收外部性 → 由社会层资质/信誉/稽核约束与运营护栏缓解。
 - 绑定（观测→动作）：当观测到异常模式（如没收率异常升高、协商接受率显著下降）时，按 §7.2 的 `SLO_T(W)` 判据与 CHG:SLO‑Runbook 执行停写/白名单/回滚等动作。

## 6. API 与事件（最小充分集）

#### 核验要点（WHY/HOW/WHAT）
- WHY：以最小接口与事件覆盖全部允许转移与会计动作。
- HOW（≤3）：
  1) ABI/事件清单与选择器一致性检查；
  2) 事件字段最小充分性（含 `via`）重放审计；
  3) 入口与事件的 E.x/INV.x 映射核对（见附录 Trace）。
- WHAT：接口/事件与 E.x/INV.x 映射一一可追溯。

### 6.1 函数（最小集）
- `createOrder(tokenAddr, contractor, dueSec, revSec, disSec) -> orderId`：创建订单，固化资产与时间锚点。
- `createAndDeposit(tokenAddr, contractor, dueSec, revSec, disSec, amount)`（payable） ：创建并立即充值指定金额（ETH：`msg.value == amount`；ERC‑20：`transferFrom(subject, this, amount)`）。
- `depositEscrow(orderId, amount)`（payable）：补充托管额，允许 client 或第三方赠与；入口遵守资产与冻结守卫。
- `acceptOrder(orderId)`：承接订单，需 `subject == contractor`，并设置 `startTime`。
- `markReady(orderId)`：卖方声明交付就绪，仅 `subject == contractor`，设置 `readyAt` 并启动评审窗口。
- `approveReceipt(orderId)`：买方验证交付并触发结清（`subject == client`）。
- `timeoutSettle(orderId)`：在评审超时后由任意主体触发全额结清。
- `raiseDispute(orderId)`：进入争议状态，`subject ∈ {client, contractor}`，记录 `disputeStart`。
- `settleWithSigs(orderId, payload, sig1, sig2)`：争议期内按签名报文结清金额 A（守卫 `A ≤ escrow`）。
- `timeoutForfeit(orderId)`：争议超时由任意主体触发对称没收。
- `cancelOrder(orderId)`：根据守卫（G.E6/G.E7/G.E11）由 client 或 contractor 取消订单。
- `withdraw(tokenAddr)`：提取累计收益或退款（Pull 语义，`nonReentrant`）。
- `extendDue(orderId, newDueSec)`：client 单调延长履约窗口。
- `extendReview(orderId, newRevSec)`：contractor 单调延长评审窗口。

### 6.2 事件（最小字段）
- `OrderCreated(orderId, client, contractor, tokenAddr, dueSec, revSec, disSec, ts)`
- `EscrowDeposited(orderId, from, amount, newEscrow, ts, via)`
- `Accepted(orderId, escrow, ts)`
- `DisputeRaised(orderId, by, ts)`
- `Settled(orderId, amountToSeller, escrow, ts, actor)`（`actor ∈ {Client, Timeout}`）
- `AmountSettled(orderId, proposer, acceptor, amountToSeller, nonce, ts)`
- `Forfeited(orderId, ts)`
- `Cancelled(orderId, ts, cancelledBy)`（`cancelledBy ∈ {Client, Contractor}`）
- `BalanceCredited(orderId, to, tokenAddr, amount, kind, ts)`（`kind ∈ {Payout, Refund}`）
- `BalanceWithdrawn(to, tokenAddr, amount, ts)`

### 6.3 授权与来源（2771/4337）
- `EscrowDeposited.via ∈ {address(0), forwarderAddr, entryPointAddr}`；`address(0)` 表示直接调用（`msg.sender == tx.origin`）。
- 2771：记录转发合约地址，且 `isTrustedForwarder(via)=true`；4337：记录配置的 `EntryPoint` 地址。
- 除上述三类外的合约调用 MUST `revert`（ErrUnauthorized）。不支持多重转发/嵌套；检测到多跳 MUST `revert`。授权失败为回滚路径，不发 `EscrowDeposited` 事件。

#### 核验要点（HOW/WHAT）
- HOW（≤3）：
  1) 构造“多跳转发”交易，预期回滚（回执 ABI 解码为错误 `ErrUnauthorized`）；
  2) 扫描该交易日志，确认未出现 `EscrowDeposited` 事件（按 ABI 事件主题核对）；
  3) 直连/2771/4337 正常路径分别触发 `via` 字段（`address(0)`/`forwarderAddr`/`entryPointAddr`）。
- WHAT：多跳检测可通过回执/事件复核；授权失败无资金变动且无事件。

#### 定义与可观测锚点（补充，不改变语义）
- “多跳（multi‑hop）”界定：在同一笔调用中，经由多个转发/中继合约层层转发至本合约的情形（除受信任的 `forwarder` 或配置的 `EntryPoint` 之外的额外一跳及以上），均视为“多跳”。
- 判定口径：若 `msg.sender` 为合约地址，且既不是受信任的 `forwarder` 也不是配置的 `EntryPoint`，则视为未授权；若检测到“多跳”链路，MUST `revert`（`ErrUnauthorized`）。
- 可选工件（建议）：
  - 自定义错误签名：`ErrUnauthorized()`（或 `ErrUnauthorized(address caller, address via)`）；
  - 事件主题示例：`EscrowDeposited(uint256 orderId, address from, uint256 amount, uint256 newEscrow, uint256 ts, address via)` 的主题 `keccak256` 值；外部审计据此核对“未发事件”。

## 7. 可观测性与 SLO（公共审计）

#### 核验要点（WHY/HOW/WHAT）
- WHY：给出可复现的指标口径与 SLO 判据，支撑公共审计与回退。
- HOW（≤3）：
  1) 以事件时间戳/字段计算度量（见计数与去重）；
  2) 按 `SLO_T(W)` 判定是否触发停写/白名单/回滚；
  3) 记录窗口/阈值/来源于 CHG 工件。
- WHAT：指标与判据一一可判定，触发动作有据可查。

### 7.1 指标（定义/单位/窗口）
- MET.1 结清延迟 P95、超时触发率。
- MET.2 提现失败率、重试率。
- MET.3 资金滞留余额。
- MET.4 协商接受率 = `#AmountSettled / #DisputeRaised`（每单仅计首个 `AmountSettled`）。
- MET.5 零协议费违规计数：期望=0（来源：回执内 `ErrFeeForbidden` 回滚次数）。
- GOV.1 终态分布（成功/没收/取消）。
- GOV.2 `A/E` 基线分布（以进入 Reviewing/Disputing 时的 E 为基线）。
- MET.6 状态转换延迟/吞吐（E1/E3/E5/E10 等非终态转换的时延与速率）。

#### 计数与去重规则（口径约束）
- `DisputeRaised`：同一订单在结清/没收前的首次计 1 次；重复触发/撤销不重复计数。
- `AmountSettled`：每单仅计首个；重复上链不重复计数。
- 可重复事件（订单维度）：`EscrowDeposited`（允许多次充值）、`BalanceWithdrawn`（按余额领取节奏可多次）。
- 其它事件：每订单计一次（`OrderCreated/Accepted/Settled/Forfeited/Cancelled/BalanceCredited`）。

#### 新增：GOV.3 争议时长
- 定义：从 `DisputeRaised.ts` 到终态 `Settled/Forfeited.ts` 的持续时间；
- 单位/窗口：秒；按窗口聚合（P50/P95/直方）；
- 说明：Forfeited 与 Settled 均纳入；取消不计。

### 7.2 SLO 与回滚剧本
- 判据：`SLO_T(W) := (MET.5=0) ∧ (forfeit_rate ≤ θ) ∧ (acceptance_rate ≥ β) ∧ (p95_settle ≤ τ)`；`θ/β/τ` 与窗口 `W` 由部署侧在 CHG 定义。
- 动作：超阈触发“切换白名单/停写/回滚”剧本；退出条件模板：若【唯一判据】在窗口 `W` 内持续满足，则执行【停止/退出/回滚】（默认 UTC）。

#### 分析方法（信息性，不进入守卫）
- 单调性检验、分位/秩回归、断点/DiD、离群点告警；用于分析与告警，不改变合约语义。

## 8. 版本化与变更管理

### 8.1 语义版本
- 状态机/不变量/API/指标任一变更 → 次版本以上；破坏性变更 → 主版本。

### 8.2 变更卡（CHG）
- 记录影响面（状态机→接口/事件→不变量→指标链路）、迁移/回滚步骤、兼容窗口。

### 8.3 兼容别名（信息性）
- 为降低集成断裂，提供一个小版本周期别名支持：`acceptBySig`（→`settleWithSigs`）、`PriceSettled`（→`AmountSettled`）。
- 别名仅作为入口别名（不重复发事件），在下一个次版本周期移除；ABI/SDK 应同时暴露别名并标注 deprecated；遥测/日志可区分别名与规范入口（记录函数选择器/方法名）。

## 9. 结果与性质（博弈观点，信息性）

### 9.1 定义（R1–R4）
- R1 付款不劣：`E − A ≥ 0`；无争议 A=E；协商 A≤E。
- R2 没收为劣（常见条件）：若 `R(t) < A`，则卖方 `A − R(t) > 0 ≥ −I(t)`；买方 `V−A ≥ −E`。
- R3 均衡路径存在：存在 `A ∈ (C, min(V,E)]` 且 `κ(Review/Dispute)=1` 时，“交付并结清”为子博弈完美均衡路径。
- R4 Top‑up 单调：`E` 单调增加把买方偏好由 `forfeit/tie` 推向 `pay`。

### 9.2 证明草图与观测锚点
- R1：由 INV.1–3 与 `A≤E` 直接得出；观测：`Settled/Balance{Credited,Withdrawn}`。
- R2：对卖方 `A−R(t)` 相对没收 `−I(t)` 优；观测：`GOV.1` 没收率与 `MET.6` 时延分布。
- R3：窗口有界与 `κ=1` 支撑有限步终止；观测：最长时长 ≤ `D_due+D_rev+D_dis` 的路径覆盖。
- R4：`E` 通过 `depositEscrow` 单调增加，协商在 `A ≤ E` 守卫下进行；观测：`GOV.2` 的 `A/E` 分布与 `EscrowDeposited` 曲线。

### 9.3 反例库（信息性）
- Ex‑1（破坏 A≤E 或允许减少 E）：R1 不成立，金额口径失真。
- Ex‑2（允许延长 D_dis 或引入外部裁决）：有限终止被破坏，均衡性质失效。
- Ex‑3（签名域缺项/重放）：跨单/跨链重放破坏价格路径与会计。

反例映射（信息性）：
- Ex‑1 → R1；Ex‑2 → R3；Ex‑3 → R4。

## 10. 有效性判据与参数（Effectiveness）

#### 核验要点（WHY/HOW/WHAT）
- WHY：将性质/指标/对照整合成单一有效性谓词，支撑运行门槛与回退。
- HOW（≤3）：
  1) 评估 R1–R4（以事件/度量为证据）；
  2) 计算 `SLO_T(W)` 与 `Δ_BASELINE(W)`（`f/数据源/窗口` 由 CHG 工件绑定）；
  3) 依据结果执行运行手册动作（停写/白名单/回滚）。
- WHAT：`Effectiveness(W)` 可单点判定（True/False）。

### 10.1 判定谓词
- `Effectiveness(W) := (R1 ∧ R2 ∧ R3 ∧ R4) ∧ SLO_T(W) ∧ Δ_BASELINE(W) ≥ 0`。

### 10.2 统一参数表
- `W`：观测窗口（如 7/14/30 天）
- `θ`：没收率上限（`forfeit_rate ≤ θ`）
- `β`：协商接受率下限（`acceptance_rate ≥ β`）
- `τ`：结清 P95 延迟上限（`p95_settle ≤ τ`）
- `f`：对照加权函数（对 succ/forf/p95/acc 的加权）

### 10.3 判定流程
- 先验收 `SLO_T(W)`，再计算 `Δ_BASELINE(W)`，最后核对 R1–R4 的观测证据。

## 11. 基线对照与适用性（Baselines & Applicability）

### 11.1 对照口径（相同窗口/来源/字段）
- `succ = #Settled / (#Settled + #Forfeited + #Cancelled)`
- `forf = #Forfeited / (#Settled + #Forfeited + #Cancelled)`
- `p95_settle`（按指标定义）
- `acc`（按指标定义）
- `Δ_BASELINE(W) = f(succ, forf, p95_settle, acc)_NESP − f(…)_Baseline`

### 11.2 选择指引
- 倾向 NESP：主观性交付/一次性交换，需最小内核/公共审计/零协议费；需要 `A∈[0,E]` 的部分结清与对称威慑；可接受“限时协商 + 对称没收”。
- 倾向中心化托管/仲裁（类外）：复杂证据/强监管/KYC、可接受平台费与裁量。
- 其他：原子交换/客观条件 → HTLC；高频低争议 → 状态通道；投票裁决可接受 → 去中心化仲裁（类外）。

## 12. 工程实现与安全护栏（Engineering & Safety）

### 12.1 最小工程实现清单
- 接口/事件/错误：采用第 6 章最小充分集。
- 签名域绑定：订单/资产/数额/截止/链标识/随机数；跨单/跨链/过期/域错路径测试。
- Pull/CEI/授权/重入：提现前清零、`nonReentrant`、授权校验与来源记录。
- 非标资产：由适配层与白名单策略处理；异常资产路径显式失败。

### 12.2 测试与验证（代表性）
- 三类用例：无争议全额/签名金额/超时没收。
- 属性：A≤E、时间窗单调/有界、签名域正确、Pull/CEI 安全、事件字段完备、ForfeitPool 语义一致。

## 13. 分阶段开放与治理（Phased Opening & Governance）

- 渗透度三档（低/中/高）：观测重点、门槛字段（W/θ/β/τ）、开放动作。
- 开桥动作（应用层）：额度/清算/缓冲/熔断等均在应用层实施，不改变合约内核。
- 参数与变更：用版本化记录窗口/阈值/权重与对照数据源；提供变更卡与回退剧本。

## 14. 风险与残余（Risks & Residuals）

- 行为与外溢：破坏型对手、策略波动、协商失败导致的外部性（高没收期）。
- 技术与运行：MEV/排序、非标资产精度/费率、代付路径的授权/来源伪装。
- 缓解与回退：护栏（授权/冻结/时间窗）、SLO 阈值与停写/白名单/回滚动作、审计与复现。

## 15. 运行与对照的变更卡绑定（CHG 必须项）

- Effective‑Params：`{ W, θ, β, τ, f, version, updated_at }`
- Baseline‑Data：`{ source, fields_map, window=W, version }`
- SLO‑Runbook：`{ thresholds_ref, runbook_uri, rollback_steps, contacts }`

## 16. 附录（Appendices）

### 16.1 术语与符号表（选摘）
- `E` 托管额；`A` 结清额；`V` 买方价值；`C` 卖方成本；
- `D_due/D_rev/D_dis` 履约/评审/争议窗口；`startTime/readyAt/disputeStart` 锚点；
- `ForfeitPool` 罚没逻辑账户（不对外分配）。

### 16.2 指标与事件口径表（简）
- 事件：`OrderCreated/EscrowDeposited/Accepted/DisputeRaised/Settled/AmountSettled/Forfeited/Cancelled/Balance{Credited,Withdrawn}`。
- 指标：MET.1/2/3/4/5/6 与 GOV.1/2/3；窗口/来源版本化（CHG）。

### 16.3 Trace 示意（状态→接口/事件→不变量→指标）
- 示例 1：E4（Executing→Settled，approveReceipt） → `Settled` → INV.1 → MET.1/MET.3。
- 示例 2：E12（Disputing→Settled，settleWithSigs） → `AmountSettled, Settled` → INV.2 → MET.4。
- 示例 3：E13（Disputing→Forfeited，timeoutForfeit） → `Forfeited` → INV.8 → GOV.1/GOV.3（争议时长）。

### 16.4 目标与条款对照（WHY→WHAT）
说明：列出关键目标（WHY）与正文中的可判定条款/守卫/指标（WHAT），用于自检各目标是否得到覆盖。
- 可信中立/零协议费 → INV.14 零费恒等式；错误 `ErrFeeForbidden`；事件/会计可重放
- 有限步终止 → D_dis 固定不可延长；E13 `timeoutForfeit`；时间边界 §5.4
- 可复现审计 → 事件最小字段 §6.2（含 `via`）；指标与去重 §7.1；SLO 判据 §7.2
- A≤E 与 Pull → INV.1/INV.2/INV.10；`withdraw` 幂等 §4.2；会计事件 `BalanceCredited/Withdrawn`
- 授权与来源可观测 → 授权/来源 §6.3；多跳回执/不发事件（ErrUnauthorized）

## 版权与许可
- 零协议费承诺适用于合约层；本白皮书遵循仓库同目录下 `LICENSE` 文件所载许可条款。
