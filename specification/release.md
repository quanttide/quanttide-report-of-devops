# Release 发布管理 — 领域建模报告

## 1. 核心概念

| 概念 | 类型 | 说明 |
|------|------|------|
| ReleaseStatus | 枚举 | 版本所处的生命周期状态，共 4 种 |
| ReleaseEntry | 事件 | journal 中的一行，不可变、仅追加，记录一次状态变更 |
| ReleaseRecord | 投影 | 内存中的业务实体，由 journal 回放投影得出 |
| Transition | 行为 | 状态间的合法转换，由 CLI 命令触发 |
| Journal | 存储 | 追加写的 JSONL 事件日志，是系统唯一事实来源 |

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

## 3. 事件与投影

### ReleaseEntry（journal 事件，不可变）

```
ReleaseEntry {
    id:         String    — UUID，同一次尝试内所有 entry 共享
    version:    String    — vX.Y.Z 或 pkg/vX.Y.Z
    status:     ReleaseStatus
    created_at: String    — 该事件被记录的时间（UNIX 秒）
}
```

### ReleaseRecord（业务实体，由回放得出）

```
ReleaseRecord {
    id:         String    — UUID，取最后一个 entry 的 id
    version:    String
    status:     ReleaseStatus  — 取最后一个 entry 的 status
    created_at: String    — 取第一个 entry 的 created_at（首次 stage 时间）
    updated_at: String    — 取最后一个 entry 的 created_at（最近变更时间）
}
```

### 投影示例

```
Journal:                                            Record:
{id:u1, v1.0.0, Staged,     created_at:100}        {id:u2, v1.0.0, Published,
{id:u1, v1.0.0, Cancelled,  created_at:200}          created_at:100, updated_at:400}
{id:u2, v1.0.0, Staged,     created_at:300}
{id:u2, v1.0.0, Published,  created_at:400}
```

- `created_at` = 100（第一个 entry，首次 stage）
- `updated_at` = 400（最后一个 entry，本次发布）
- `id` = u2（最后一个 entry，即当前尝试）
- 版本号是业务标识键。`FileStorage::load(version)` 按 version 查找。

## 4. 持久化：事件溯源模式

### 存储路径

`.quanttide/devops/release-journal.jsonl`

### 读写策略

- **写** (`save`)：将 `ReleaseRecord` 转成 `ReleaseEntry`（`created_at = record.updated_at`），作为一行 JSON 追加到文件末尾
- **读** (`new`)：顺序回放所有行，解析为 `ReleaseEntry`，按 version 投影为 `Vec<ReleaseRecord>`。投影规则：`created_at` 取该 version 的第一个 entry 的时间，`updated_at`/`status`/`id` 取最后一个 entry

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
| ReleaseRecord | 1 | 字段值正确性 |
| FileStorage | 7 | 读写、不存在、更新（含 updated_at 变更）、journal 追加、list、跨实例持久化、created_at 跨更新保持 |
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

6. **事件与投影分离**：`ReleaseEntry`（journal 行，不可变）与 `ReleaseRecord`（业务实体，投影得出）是两种不同的模型。Entry 只有 `created_at`（事件发生时间），Record 有 `created_at`（首次 stage）和 `updated_at`（最近变更）。分离后 journal 的不可变语义更清晰，record 的字段含义也不被序列化格式污染。

7. **`created_at` 跨尝试保持**：`created_at` 记录的是该 version 首次 stage 的时间，即使被 cancelled 后 restage（新 UUID）也不会重置。这与 event sourcing 的回放逻辑一致：投影时取该 version 所有 events 中最早的 `created_at`。
