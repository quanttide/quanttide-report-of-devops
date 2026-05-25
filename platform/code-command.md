# code 命令修复报告

基于 `qtcloud-devops-cli v0.3.0` 实际使用体验整理。所有问题已在 v0.3.x 迭代中修复。

## 修复项

| 优先级 | 问题 | 方案 | 状态 |
|--------|------|------|------|
| P0 | 状态误判 | 修正状态判定：工作区脏才标 Dirty，父指针落后标 AheadOfParent | ✅ `src/model/code.rs` |
| P0 | remote_head 是本地缓存 | status 默认先 fetch，失败时降级到本地缓存并标记 🛰 | ✅ `src/commands/code.rs` |
| P1 | `--dry-run` 位置 | 下放到 `sync` / `retire` 各子命令 | ✅ `src/main.rs` |
| P2 | 输出冗余 | 改为单行聚合格式 | ✅ `src/commands/code.rs` |
| P3 | 失败提示 | 引入颜色和 `✓` / `✗` 状态标记 | ✅ `src/commands/code.rs` |
| P4 | 离线场景 | 增加 `--offline` 参数跳过 fetch | ✅ `src/main.rs` |

## 拒绝标准

- [x] Dirty 列表不再包含纯 AheadOfParent 的子模块
- [x] status 默认执行 fetch，远程有新 push 时正确显示 BehindRemote
- [x] `code sync --dry-run` 通过编译并能正常执行 dry-run
- [x] 17 个子模块输出在 20 行以内，失败项目显式标记
