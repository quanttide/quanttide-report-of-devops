# ADR-002: git 网络操作用 CLI 而非 git2

- **状态**: 已采纳
- **日期**: 2026-07-06
- **涉及项目**: qtcloud-devops

## 背景

`release publish` 在推送 tag 到 GitHub 时失败，错误为：

```
remote authentication required but no callback set
```

原因是 `git2::Remote::push()` 未配置 credential callback，无法处理 GitHub HTTPS 的 401 认证挑战。系统 `git` 命令通过 `gh auth setup-git` 配置的 credential helper 正常认证。

## 决策

任何需要网络的 git 操作必须用 `std::process::Command::new("git")`，不用 `git2` crate。

| 操作 | 原来 | 现在 |
|------|------|------|
| `push_tag` | `git2::Remote::push()` | `git push origin <refspec>` |
| `delete_remote_tag` | `git2::Remote::push()` | `git push --delete origin <tag>` |

`git2` 的使用范围：仅限纯本地只读操作（读配置、查日志、创建本地 tag、rev-parse 等）。

## 后果

- 消除"测试通过、实跑失败"的不一致
- 所有网络 git 操作继承用户终端配置的 credential helper
- 新增网络 git 操作时直接使用 `Command`，不嘉偶人 `git2`
