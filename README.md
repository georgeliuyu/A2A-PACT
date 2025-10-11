# NESP：A2A 无仲裁托管结算协议（No‑Arbitration Escrow Settlement Protocol）

副题：Trust‑Minimized · Timed‑Dispute · Symmetric‑Forfeit · Zero‑Fee

说明：本仓库承载 NESP 的白皮书、EIP 草案、参考实现规划与示例资料。白皮书为唯一规范性文档（SSOT），其语义与 EIP 草案保持等价。

- 规范（SSOT，白皮书）：SPEC/nesp-whitepaper.md
- EIP 草案（英文）：EIP-DRAFT/eip-nesp.md
- 示例（端到端与 TypedData）：EXAMPLES/minimal-flow.md
- 讨论/研究：docs/zh/NESP_协议深度研究报告.md

目录结构（目标形态）
```
nesp/
├─ README.md                    # 项目总览 + 链接
├─ SPEC/
│  ├─ nesp-whitepaper.md       # 白皮书主文（规范，SSOT）
│  └─ diagrams/                # PNG/SVG 或 Mermaid 源文件
├─ EIP-DRAFT/
│  └─ eip-nesp.md              # 按 EIP-1 模版撰写的 ERC 草案
├─ CONTRACTS/                  # 参考实现（占位，无代码）
│  ├─ NESP.sol
│  └─ interfaces/INESP.sol     # 最小接口（事件/错误/结构体）
├─ TESTS/                      # 测试配置与样例（占位，无代码）
│  ├─ Foundry.toml / hardhat.config.ts
│  └─ NESP.t.sol / NESP.spec.ts
├─ EXAMPLES/
│  ├─ minimal-flow.md          # 端到端示例 + 事件口径
│  └─ settlement_typed_data.json
├─ LICENSES/
│  ├─ CC0.txt                  # EIP 文本用
│  └─ MIT.txt                  # 代码用（或 Apache-2.0）
└─ SECURITY.md                 # 风险模型与披露窗口
```

贡献指南
- 任何影响协议语义的改动，首先更新白皮书（SPEC/nesp-whitepaper.md），并同步 EIP 草案（EIP-DRAFT/eip-nesp.md）。
- 新增/修改事件、状态或不变式，请一并更新示例与测试占位文档。
