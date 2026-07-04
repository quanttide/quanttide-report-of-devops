# DevOps 测试工作方法论

基于 qtcloud-devops-cli 的测试实践总结。

## 分层测试策略

```
CLI 集成测试（tests/cli.rs）
  通过真实二进制运行，覆盖 main.rs 分发
  ┃
库集成测试（tests/release.rs, tests/code.rs）
  调用库函数 API，覆盖模块间协作
  ┃
单元测试（src/**/mod.rs 内的 #[cfg(test)]）
  覆盖纯函数和格式化分支
```

三层各有侧重，不互相替代：

| 层 | 覆盖目标 | 执行速度 | 维护成本 |
|----|---------|---------|---------|
| CLI 集成 | main.rs 分发、端到端流程 | 慢 | 低（测试少） |
| 库集成 | 模块协作、I/O 编排 | 中 | 中 |
| 单元 | 纯函数逻辑、格式化分支 | 快 | 高（测试多） |

## 可测试性优先于测试覆盖率

加测试之前先让代码**能被测试**：

```rust
// ❌ 不可测试：直接写 stdout
pub fn status() {
    println!("结果: {}", data);
}

// ✅ 可测试：注入 writer
pub fn status_to(writer: &mut impl Write) -> io::Result<()> {
    writeln!(writer, "结果: {}", data)?;
}

pub fn status() {
    let _ = status_to(&mut std::io::stdout());
}
```

**判断标准**：纯函数（无 I/O、无外部状态）→ 单元测试；编排/I/O → 集成测试覆盖。

非纯函数如果拆出纯函数后只剩薄薄的调用层，不单独测——让集成测试间接覆盖就够了。

## 覆盖率数字只是结果

覆盖率的真正价值不是数字本身，而是**测试缺口分析**：

```
87.3% → 哪 12.7% 没覆盖？
  main.rs 19%       → CLI 分发，加集成测试
  doctor.rs 72%     → 新增代码，补单元测试
  test.rs 82%       → run() 执行外部命令，难测
```

每个覆盖缺口对应一个决策：补测试、重构、还是接受。

## 集成测试的杠杆效应

一个 CLI 集成测试可以覆盖 `main.rs` 中数十行分发代码：

```rust
// 一个测试覆盖整个 Commands::Status 分支
#[test]
fn test_cli_status() {
    let output = cli().arg("status").current_dir(d.path()).output().unwrap();
    assert!(output.status.success());
}
```

CLI 集成测试的优先级：
1. 每个命令一个 `--help` 测试（触发 clap 解析器）
2. 每个命令一个 `status` 测试（触发业务逻辑）
3. 关键路径的成功/失败测试
4. mock 外部命令的异常测试

## 外部命令的测试

命令执行代码（`Command::new("cargo").args([...])`）天然难以单元测试。处理方式：

```rust
// 测试纯函数（命令映射）
fn test_command(lang: &Language) -> Option<(&str, &[&str])> {
    // 不执行命令，只返回命令名和参数
}
// → 可单元测试

// 跳过执行层（命令执行）
fn run_tests_for_lang(dir: &Path, lang: &Language) -> Result<(), String> {
    let (cmd, args) = test_command(lang)?;
    Command::new(cmd).args(args).current_dir(dir).status()?;
    // → 不可单元测试，靠集成测试间接覆盖
}
```

**判断标准**：输出函数（返回格式化字符串）单独测；执行函数（跑命令）不单独测。

## 覆盖率的递减效应

从 0 到 80% 容易，从 80% 到 90% 需要重构，从 90% 到 95% 需要专门投入。

经验阈值：

| 区间 | 投入 | 典型缺口 |
|------|------|---------|
| < 80% | 补基本测试 | 主要模块无测试 |
| 80-90% | 重构提可测试性 | println → writeln、模块拆分 |
| 90-95% | 加集成测试 | main.rs 分发、I/O 编排 |
| > 95% | 边界值和错误路径 | 特定分支未覆盖 |

不要在 95% 以上过度投入。接受 CLI 入口代码和外部命令执行代码的低覆盖率。

## 测试纪律

1. **新增代码必须带测试** — 单元测试或集成测试，按上述分层判断
2. **纯函数值得单独测试** — 测试成本低、收益高
3. **非纯函数不要单独测** — 让集成测试间接覆盖
4. **输出格式化必须可测试** — 用 `_to(writer)` 模式
5. **外部命令映射是纯函数** — 提取出来即可单元测试
