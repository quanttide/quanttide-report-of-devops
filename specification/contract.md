# 契约体系实现对照

四维架构在 `qtcloud-devops` 工具链中的落地情况。

| 维度 | 理论定义 | 当前实现 | 状态 |
|------|---------|---------|------|
| Stages | 价值流的节拍 | `docs/handbook/lifecycle/` 八阶段导航 | ✅ |
| Platforms | 外部治理载体 | GitHub（分支保护、PR 审批、Actions） | ✅ |
| Sources | 事实源 | git tag 作为版本事实源，`release status` / `publish` | ✅ |
| Scopes | 规则的应用边界 | `contract.yaml` 定义作用域（tag 前缀 → 子目录映射） | ✅ |

四个维度均有实际代码落地，并非纯理论推演。其中 Scopes 维度在 2026 年 6 月的重构中才被重新发现并明确对应——理论设计早于实现，实现反过来验证了理论。
