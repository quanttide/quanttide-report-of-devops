# qtcloud-devops 与 qtcloud-code 的分工边界

- **状态**: 已采纳
- **日期**: 2026-07-06
- **涉及项目**: qtcloud-devops, qtcloud-code

## 背景

`qtcloud-devops`（DevOps 生命周期协调）和 `qtcloud-code`（代码静态分析）都能做代码审计。需要明确边界以避免重复造轮子和依赖膨胀。

## 决策

| 层级 | 工具 | 做什么 | 判断标准 |
|------|------|--------|----------|
| **门禁** | `qtcloud-devops code audit` | 文本级统计 | 红/绿，CI 阻断 |
| **诊断** | `qtcloud-code review` | AST 级分析 | 精确到行号，给出修复建议 |

边界线：**是否需要 parser**。devops 的新指标采纳门槛是"能否在不引入 tree-sitter 的前提下实现"。

## 联动机制

`qtcloud-code review . --status` 输出 `STATUS.md`，`qtcloud-devops code audit` 读到就聚合展示。两者独立发布、独立演进，不互相依赖做门禁判定。

## 后果

- devops 不做圈复杂度、长函数等 AST 分析
- code 不负责 CI 阻断
