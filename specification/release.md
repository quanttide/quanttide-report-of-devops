# Release 发布管理 — 领域建模报告

## 1. 核心概念

| 概念 | 类型 | 说明 |
|------|------|------|
| ReleaseStatus | 枚举 | 版本所处的生命周期状态，共 4 种 |
| ReleaseAttempt | 实体 | 一次发布尝试，以 UUID 唯一标识 |
| Transition | 行为 | 状态间的合法转换，由 CLI 命令触发 |
| Journal | 存储 | 追加写的 JSONL 事件日志，是系统事实来源 |

## 2. 状态机

```
                ┌──────────┐
         ┌───── │  Staged  │ ◄──────┐
         │      └────┬─────┘        │
         │           │              │
         │      ┌────▼─────┐   ┌────┴─────┐
         │      │ Published│   │ Cancelled │
         │      └────┬─────┘   └──────────┘
         │           │
         │      ┌────▼─────┐
         └──────│  Retired  │  (终态)
                └──────────┘
```

### 合法转换

| from → to | 命令 | 前置条件 |
|-----------|------|---------|
| (新建) → Staged | `stage` | 版本未 Published / 未 Retired |
| Staged → Published | `publish` | 必须 Staged |
| Staged → Cancelled | `cancel` | 必须 Staged |
| Cancelled → Staged | `stage` | 无（视为重新发布，生成新 UUID） |
| Published → Retired | `retire` | 必须 Published |

### 非法转换

从 `Retired` 出发的任何转换、`Published → Cancelled`、`Cancelled → Published`、`Staged → Retired` 等均被模型拒绝。

## 3. 实体：ReleaseAttempt

```
ReleaseAttempt {
    id:         UUID v4    — 每次 stage 生成（cancelled 后 restage 生成新 ID）
    version:    String     — vX.Y.Z 或 pkg/vX.Y.Z
    status:     ReleaseStatus
    created_at: u64        — UNIX 时间戳
    updated_at: u64        — 最后变更时间
    reason:     String     — 原因备注（可选，暂未从 CLI 暴露）
}
```

版本号是业务标识键。`FileStorage::load(version)` 用 version 查找，保证每个 version 在 journal 中只有一条当前状态。

## 4. 持久化：事件溯源模式

### 存储路径

`.quanttide/devops/release-journal.jsonl`

### 读写策略

- **写**：每次状态变更，将完整 `ReleaseAttempt` 作为一行 JSON 追加到文件末尾
- **读**：启动时顺序回放所有行，按 version 去重（后出现的覆盖先出现的），重建内存中的 `Vec<ReleaseAttempt>`

### 设计取舍

| 方案 | 优点 | 缺点 |
|------|------|------|
| 快照文件 + 事件日志 | 读快，审计全 | 两个文件可能不一致，复杂度增加 |
| 单文件事件溯源（选） | 单一事实来源，审计即存储，实现简单 | 每次启动需回放全部事件（数据量极小时可忽略） |

选事件溯源的原因：release 操作低频（每个版本几次），事件总量极少，回放成本可忽略。换来的是不存在"快照和日志不一致"的问题。

## 5. CLI 命令作为状态转换触发器

```
stage   -v <version>    → 新建/刷新 ReleaseStatus::Staged
publish -v <version>    → Staged → Published（执行 tag + GitHub Release）
cancel  -v <version>    → Staged → Cancelled（清理 tag + GitHub Release）
retire  -v <version>    → Published → Retired
```

CLI 参数只暴露不可推断的业务要素（版本号、是否跳过确认）。约定的东西不暴露给 CLI：
- CHANGELOG.md 固定在项目根目录
- --reason 暂不暴露（纯审计字段，与核心流程无关）

## 6. 与 Git/GitHub 的交互

状态转换到 `Published` 时会触发外部操作：

```
tag_ok = create_tag(version)        → git tag <version>
push_ok = push_tag(version)         → git push origin <version>
release_ok = create_release(...)    → gh release create ...
```

任一步骤失败则回滚（`rollback_tag`）：删除本地和远程 tag。回滚后应用状态**不**回退——用户需重新 `publish`。

`Cancel` 也会尝试删除远程 tag 和 GitHub Release（忽略错误，不做回滚）。

## 7. 模型验证（67 tests）

| 测试范围 | 数量 | 内容 |
|----------|------|------|
| ReleaseStatus | 2 | Debug、Clone+Eq |
| validate_transition | 2 | 合法 4 条 + 非法 8 条 |
| ReleaseAttempt | 2 | 新建字段、ID 唯一性 |
| FileStorage | 6 | 读写、不存在、更新、事件追加、list、跨实例持久化 |
| TransitionError Display | 1 | 错误信息 |
| stage | 6 | 新建、非法版本、已发布拒绝、cancelled restage、retired 拒绝、幂等刷新 |
| publish | 3 | 非 Staged 拒绝、不存在拒绝、用户取消 |
| cancel | 3 | 非 Staged 拒绝、不存在拒绝、正常 cancel |
| retire | 3 | 非 Published 拒绝、不存在拒绝、正常 retire |
| release utils | 25 | validate_version（6）、normalize_version（5）、precheck_changelog（6）、extract_notes（7）、confirm_release（2）、parse_github_repo（7）、create+rollback（1） |

## 8. 关键建模决策记录

1. **状态机而非过程式**：旧 `release` 命令是线性脚本，无法表达中间态。状态机将发布流程显式建模为有限状态转换，非法操作被类型系统拒绝。

2. **Retired 是终态**：不允许从 Retired 出发的任何转换。退役的版本需要走 hotfix 发新版本。

3. **Cancelled restage 生成新 UUID**：虽然 spec 有争议（"复用旧 attempt 还是生成新 UUID"），生成新 UUID 提供更清晰的审计追溯——每次发布尝试有独立身份。

4. **事件溯源而非快照**：释放了"快照 vs 日志"的一致性问题，但增加了启动回放成本。适用于低频操作场景。

5. **CLI 极简**：只暴露版本号。约定的东西（CHANGELOG 位置、存储路径）不进入 CLI 参数列表。
