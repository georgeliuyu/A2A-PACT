NESP 规范（中文 · 迁移自 docs/zh/nesp_spec.md）

# A2A 无仲裁托管结算协议（NESP）规范（节选）

（请以 docs/zh/nesp_spec.md 为准，后续以本路径统一维护）

English Title: A2A No‑Arbitration Escrow Settlement Protocol (NESP)

副题：信任最小化 · 限时争议 · 对称没收威慑 · 零手续费

发布状态：正式颁布版（Release） / 版本：0.1 / 发布日期：2025-09-30

概述（信息性）：
- NESP 是面向 A2A 的无仲裁托管结算协议：买方先将应付资金 E 托管至合约，卖方接单并交付；无争议一次性全额放款 E；发生分歧则在争议期内以可验证签名协商结清金额 A（A≤E），差额返还买方；逾期未合意则对称没收托管资金 E；零协议费（不含 gas）。

