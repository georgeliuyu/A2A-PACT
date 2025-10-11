# NAS 协议博弈分析（Bob：合约工程视角）—— 以规范为 SSOT，game 为参考

说明与边界
- SSOT：`docs/zh/nesp_spec.md`；本文仅以工程化语言复述博弈含义与实现护栏，不改变语义。
- 参考：`docs/zh/nesp_game_theory.md`（定理/基线对比为“信息性”）。

工程口径下的博弈核心（对齐 SSOT）
- 金额口径：`A ≤ E(t_settle)`；正常流按当前 `E` 全额结清；争议流仅验签与上限（SSOT §1/§2）。
- 时间与势函数：`D_due/D_rev` 可延后（单调）、`D_dis` 固定；结合锚点，保证有限步终止（SSOT §2/§6）。
- 对称威慑：超时进入 `Forfeited`，双方都拿不到这笔 `E`（SSOT §2/§6）。

状态机与事件映射（实现最小集）
- 主要入口：`createOrder/createAndDeposit/depositEscrow/acceptOrder/markReady/approveReceipt/timeoutSettle/raiseDispute/settleWithSigs/timeoutForfeit/extendDue/extendReview/withdraw`（SSOT §7）。
- 事件基线：`OrderCreated/EscrowDeposited/Accepted/Settled/AmountSettled/Forfeited/Cancelled/Balance{Credited,Withdrawn}`（SSOT §7）。
- 守卫与错误：`ErrInvalidState/ErrBadSig/ErrExpired/ErrOverEscrow/ErrUnauthorized/ErrFeeForbidden/...`（SSOT §5/§7）。

安全与不变量（把博弈假设落到代码）
- Pull 语义：状态入口不直接对外部地址 `transfer`；仅在 `withdraw*` 路径实际发币；`nonReentrant` 与 CEI。
- 签名域与重放：绑定 `orderId/tokenAddr/amount/nonce/deadline/chainid`；`nonce` 单调；过期回滚；跨订单/跨链重放无效（SSOT §5）。
- ETH 与 ERC‑20：`createAndDeposit/depositEscrow` 统一处理 ETH 与 ERC‑20，精度一致；`SafeERC20`；防“精度提升/四舍五入”攻击（向下取整）。
- 时间窗口：所有延时只能“向后”；不得延长 `D_dis`；基于 `block.timestamp`；避免区块时间抖动影响分支判定。

监测与 SLO（把博弈性质做可观测）
- 结清延迟与资金滞留（MET.1/3）：作为执行/拥塞的健康度；
- 协商接受率（MET.4）：签名金额路径的可用性；
- 终态分布与 `A/E`（GOV.1/2）：观察没收率与价格行为；
- 零协议费违规计数（MET.5）：必须为 0（回执日志对账）。

失败模式与回归点（短表）
- 签名域不一致/nonce 重放 → `ErrBadSig/ErrReplay`；
- 错误口径导致 `A > E` → 阻断（属性测试/形式化）；
- 非标准 ERC‑20 回滚/黏滞余额 → 适配层白名单；
- 代付/转发（4337/2771） → 只允许受信入口，事件 `via` 字段如 SSOT；
- 时间边界旁路（延长 `D_dis`）→ 阻断；
- 直接在入口转账 → 违反 Pull，阻断。

结论（Bob 视角）
- NAS 的博弈均衡假设通过“`A ≤ E` + 有界时间 + 对称威慑”在代码层可检与可测。保持最小内核与统一口径，优先构建属性测试与事件监测来“证伪式”保障均衡前提。

互学增补（吸收其他专家要点；不改变 SSOT）
- 来自 Vitalik（可信中立）：强制“零协议费”与禁止内置仲裁/表决；在回执日志中对 `ErrFeeForbidden` 进行离线对账，确证 MET.5=0。
- 来自 Tim（简单且近优）：在保证安全与正确性的前提下，保持接口与状态机最小化，降低实现复杂度与攻击面。
- 来自 Glen（公平与公共审计）：确保事件字段支持 `A/E` 分布与没收率统计（GOV.1/GOV.2），为第三方公共审计提供原始口径。
- 来自 Nenad（渗透性治理）：把“桥额度/清算周期/缓冲与停市熔断”置于应用层或外围网关，不在合约内核实现。
- 来自 Blade（最小验证集）：将“签名域/nonce/过期、Pull、`A ≤ E`、`D_*` 单调、代付白名单、非标准 ERC‑20 适配”纳入属性与模糊测试基线，配 SLO/回滚剧本。
