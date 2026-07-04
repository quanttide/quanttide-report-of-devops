# DevOps 测试阶段工作纪要

## 阶段成果

- 测试总数：307（+28）
- 覆盖率：87.3% → **91.1%**
- CLI 集成测试：9 → **26**

## 关键改进

### 1. 可测试性重构

`build.rs`、`test.rs`、`release/status.rs` 全部拆分为 `status_to(writer)` + `status()` 模式。输出不再硬编码 `println!`，改为 `writeln!(writer, ...)?`，通过注入 `Vec<u8>` writer 使格式化分支可单元测试。

### 2. 默认阈值 70% → 80%

所有硬编码的阈值数字改为引用 `StageTest::default()`，确保一个事实源。

### 3. 集成测试覆盖 main.rs 分发

通过 `env!("CARGO_BIN_EXE_qtcloud-devops")` 运行真实二进制，覆盖所有命令的 CLI 进入路径。从 9 个测试扩展到 26 个，每个命令至少有一个 `--help` 和一个 `status` 测试。

### 4. 语言感知诊断

`doctor status` 按项目检测到的语言只显示相关工具链，`contract status` 汇总所有语言。

### 5. 发现并修复的问题

| 问题 | 发现方式 | 修复 |
|------|---------|------|
| contract.yaml `platforms:` 字段名写错 | 审查输出 | 对齐模型字段名 |
| eprintln 暴露原始 OS 错误 | 审查输出 | 静默降级 |
| test status 重复加载 contract.yaml | 审查输出 | 复用已传入的 `c` |
| CHANGELOG 重复条目 | 自举验证 | LLM 生成+手动编辑两条路径冲突 |
| publish 超时后无重跑机制 | 自举验证 | push_tag/create_release 幂等 |
| `nth(1)` vs `lines()` 迭代器消费 | 编译错误 | 一次性迭代器注意事项 |

## 当前覆盖缺口

| 模块 | 覆盖率 | 原因 |
|------|--------|------|
| `main.rs` | 34.5% | CLI 分发代码，依赖集成测试覆盖 |
| `test.rs` run 系列 | 82.4% | `run_tests_for_lang` 等执行外部命令，测试成本高 |
| `release/status.rs` | 85.6% | 输出格式化分支多，部分需 gh mock |
| `plan.rs` | 88.7% | 规约解析逻辑复杂 |
