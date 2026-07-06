# git 库使用范围：gix / git2 / CLI

- **状态**: 已采纳
- **日期**: 2026-07-06
- **涉及项目**: qtcloud-devops

## 背景

项目依赖两个 git 库（`gix` 和 `git2`）以及系统 `git` 命令，三者在功能上有重叠。需要明确各自的使用范围，避免用错导致认证失败、测试不一致等问题。

## 决策

按操作类型选择工具：

| 操作类型 | 使用 | 示例 |
|----------|------|------|
| 只读查询（本地） | `gix`（优先） | 读 remote URL、查配置、遍历引用 |
| 本地写入 | `git2` | 创建本地 tag、删除本地引用 |
| 网络操作 | `git` CLI（`Command`） | push、fetch、pull、rebase、clone、delete remote tag |

### 原则

1. **`gix` 优先用于只读操作**——`gix` 是纯 Rust 实现，无 C 依赖，编译快，API 安全。能读配置、查 remote URL 等场景先用 `gix`。
2. **`git2` 只做本地写入**——`git2`（libgit2 绑定）有 C 依赖，编译慢，但写操作 API 成熟（创建 tag、删除引用）。
3. **网络操作一律走系统 `git` CLI**——`git2` 的 credential callback 未配置，在 GitHub HTTPS 上会报 `authentication required but no callback set`。系统 `git` 通过 `gh auth setup-git` 配置的 credential helper 正常认证，同时消除"测试通过、实跑失败"的不一致。

## 后果

- `gix` 负责的只读操作无 C 编译开销
- `git2` 降级为仅本地写入，不涉及网络
- 新增网络 git 操作时直接用 `Command::new("git")`
- 三个工具各管一摊，不交叉使用
