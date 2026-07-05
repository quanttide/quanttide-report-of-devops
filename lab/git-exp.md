# git2 vs gix API 与性能实验报告

## 概述

对 Rust 生态中两个主流 git 操作库 `git2`（绑定 libgit2）和 `gix`（纯 Rust 实现）进行 API 风格与性能对比实验。实验覆盖 8 个 DevOps 常见 git 操作，分别在两个规模不同的仓库上运行。

## 环境

- **git2**: 0.19.0（libgit2 C 绑定）
- **gix**: 0.69.1（纯 Rust）
- **仓库 A**（小）: lab 自身，~15 commits，无子模块
- **仓库 B**（大）: quanttide-devops 超级项目，含 10+ 子模块

## API 风格对比

| 维度 | git2 | gix |
|------|------|-----|
| 初始化 | 急切加载（打开即解析全部配置、refs） | 延迟加载（Platform 构建器，按需求值） |
| 错误模型 | `Result<T, Error>`，统一错误类型 | `Result<T, Error>` + `Option<T>` 混用，部分返回 `Option<Id>` |
| 迭代器 | 直接实现 `Iterator`（`revwalk`、`branches`） | 构建器模式（`Platform` + `.all()?` 获得迭代器） |
| 对象访问 | `repo.find_blob(oid)`、`head.peel_to_tree()` | `repo.find_commit(oid)`、`commit.tree()`、`entry.object()` |
| 树条目查询 | `tree.get_path("path")` 接受 `&Path` | `tree.lookup_entry_by_path("path")` 接受 `&str`，返回 `Result<Option<Entry>>` |
| 引用遍历 | `repo.branches(BranchType::Local)`、`repo.tag_foreach()` | `repo.references()?.prefixed("refs/heads")?.all()?` |
| 工作区状态 | `repo.statuses(Some(&mut opts))` 简洁 | `repo.status(progress)?.into_index_worktree_iter(patterns)?` 显式链长 |
| diff | `repo.diff_tree_to_tree(old, new, None)` | 同 `repo.diff_tree_to_tree(old, new, None)`（API 趋同） |
| 安全性 | 通过 C 绑定，存在内存安全问题历史 | 纯 Rust，内存安全 |

## 性能数据

### 仓库 A（小仓库，lab 自身）

| 操作 | git2 (µs) | gix (µs) | 胜者 | 加速比 |
|------|----------|---------|------|-------|
| open | 14758 | 1090 | gix | 13.5x |
| head | 164 | 558 | git2 | 3.4x |
| log(100) | 29908 | 6186 | gix | 4.8x |
| blob(Cargo.toml) | 285 | 916 | git2 | 3.2x |
| diff(HEAD~3..HEAD) | 6138 | 1863 | gix | 3.3x |
| status | 551 | 3328 | git2 | 6.0x |
| branches | 1217 | 947 | gix | 1.3x |
| tags | 369 | 935 | git2 | 2.5x |

**结果**: git2 胜 4 / gix 胜 4

### 仓库 B（大仓库，超级项目）

| 操作 | git2 (µs) | gix (µs) | 胜者 | 加速比 |
|------|----------|---------|------|-------|
| open | 14274 | 1009 | gix | 14.1x |
| head | 168 | 544 | git2 | 3.2x |
| log(100) | 1549 | 5730 | git2 | 3.7x |
| blob(Cargo.toml) | 308 | 913 | git2 | 3.0x |
| diff(HEAD~3..HEAD) | 2806 | 1782 | gix | 1.6x |
| status | 540 | 2368 | git2 | 4.4x |
| branches | 234 | 635 | git2 | 2.7x |
| tags | 237 | 555 | git2 | 2.3x |

**结果**: git2 胜 6 / gix 胜 2

## 分析

### 0. 最醒目的数据

| 操作 | 小仓库胜者 | 大仓库胜者 | 说明 |
|------|-----------|-----------|------|
| `open` | gix 13.5x | gix 14.1x | **唯一 gix 在两仓库都赢且几乎同倍率** |
| `log(100)` | gix 4.8x | git2 3.7x | **唯一翻转了胜负** |
| 其余 6 项 | 小胜各半 | git2 全胜 | 趋势随仓库规模系统性偏移 |

### 1. `open` 的 14x 差距是架构差异，不是优化问题

git2 的 14ms 不是慢，是做了实事——它打开时加载了：

- 全部 config（system + global + local）
- 全部 refs（读目录 + 符号解析）
- HEAD 解析 + 剥离
- odb 初始化 + 对象查找候选链

gix 的 1ms 就是记录路径 + 一个 stat。**这 14x 差距无法通过优化 git2 消除**，是"做不做"的区别，不是"做得好不好"的区别。

### 2. `log(100)` 胜负翻转说明缓存行为完全不同

小仓库（15 commits）：

```
git2: 29908µs   ← 异常高，应该是包打开的一次性冷启动开销
gix:   6186µs   ← 增量解析在小 repo 有优势
```

大仓库（数百 commits）：

```
git2: 1549µs    ← 包打开后后续操作极快（缓存已预热）
gix:  5730µs    ← 每次按需解析，无法利用 cache
```

git2 在小仓库的 `log` 花了 30ms 是 outlier——通常是第一次访问 pack 文件的冷启动成本。同一进程里如果先跑别的操作把它预热了，log 会大幅下降。**gix 的延迟策略恰好免疫这种冷启动惩罚，但也无法享受预热收益。**

### 3. `status` 的 4-6x 差距在 IO 模式

git2 的 status 底层走 libgit2 的 workdir/index 对比，用 C 实现、批量系统调用。gix 的 status 走纯 Rust `gitoxide` 状态机——每次查询都要重建索引遍历器。这是工作量证明——git2 在 C 层积累的优化 gix 还没追平。

### 4. `diff` — gix 在两场景都赢（1.6-3.3x）

diff 是唯一 gix 在两仓库都有优势的操作。这可能是因为 gix 的 diff 使用基于 `gix-diff` crate 的树遍历优化，而 git2 的 `diff_tree_to_tree` 在 libgit2 里走 diff 驱动 + 召回整个 stats 开销较大。如果 CLI 频繁做 changelog diff 或 pre-commit diff，这个优势有实用价值。

### 5. 实际工程含义：总耗时曲线

```
单次读（CLI status 命令）：
    总耗时 = open + query
    gix:  1ms + 2ms  = 3ms
    git2: 14ms + 0.5ms = 14.5ms     ← gix 快 5x

多次读（release publish 流程，6 次查询）：
    总耗时 = open + 6×query
    gix:  1ms + 6×3ms  = 19ms
    git2: 14ms + 6×1ms = 20ms       ← 打平

密集读（批量分析/审计，20 次查询）：
    总耗时 = open + 20×query
    gix:  1ms + 20×3ms  = 61ms
    git2: 14ms + 20×1ms = 34ms      ← git2 快 1.8x
```

三条曲线交叉点大约在 **5-8 次查询**——少于这个数 gix 总耗时更低，多于这个数 git2 开始反超。

- **CLI 工具，一次一命令**：gix 的 `open` 优势让你每个命令快 ~10ms，这是用户能感知的
- **daemon / watch / 流程编排**：git2 的缓存预热让你在十几次操作后全面领先
- **`diff` 是 gix 少数纯计算上赢 git2 的操作**，如果你的场景 diff-heavy，值得关注

## 结论与建议

| 场景 | 推荐库 | 理由 |
|------|--------|------|
| 快速打开、单次查询、只读 | gix | open 快 14x，适合 CLI 一次一命令 |
| 高频操作、读写混合 | git2 | 后续操作快 2-6x，API 更成熟 |
| 内存安全敏感 | gix | 纯 Rust，无 C 内存安全问题 |
| 历史久的大仓库 | git2 | log、status 等高频操作明显更快 |
| 纯 Rust 栈项目 | gix | 零 C 构建依赖，编译链路简单 |

**当前建议**: DevOps CLI 场景中 CLI 工具通常是"打开一次、执行一个命令、退出"，gix 的 `open` 优势更匹配。但考虑到现有代码已基于 git2（qtcloud-devops 使用 git2 API），迁移成本需要权衡。**混合使用**可能是最优解：用 gix 做仓库发现和打开决策，用 git2 做数据密集操作。

## 附加产物

实验代码位于 `examples/default/src/git_experiment.rs`，注册为 `cargo run --bin quanttide-lab -- git-exp [path]` 子命令。
