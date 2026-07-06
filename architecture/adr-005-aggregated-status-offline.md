# ADR-005: 聚合 status 默认离线模式

- **状态**: 已采纳
- **日期**: 2026-07-06
- **涉及项目**: qtcloud-devops

## 背景

顶层 `qtcloud-devops status` 命令聚合 contract / source / plan / code / build / test / release 七个模块的状态。其中 `code::status` 默认 online 模式，逐个连接 18 个子模块的 remote 查询 ahead/behind，导致命令长时间卡住。

## 决策

顶层 `status` 命令向 `code::status` 传入 `offline: true`，不联网。

```rust
// main.rs
if let Ok(report) = qtcloud_devops_cli::code::status(rp.clone(), true) {
```

## 使用场景

| 命令 | 默认 | 适用场景 |
|------|------|----------|
| `status`（顶层聚合） | offline | 快速概览所有模块 |
| `code status` | online | 专门查看子模块同步状态 |

`code status --offline` 可用于离线场景。

## 后果

- 顶层 status 秒出，不再因网络卡顿
- 用户如需查看 ahead/behind 精确计数可单独运行 `code status`
