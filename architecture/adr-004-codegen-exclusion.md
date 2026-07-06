# ADR-004: codegen 文件跳过规则

- **状态**: 已采纳
- **日期**: 2026-07-06
- **涉及项目**: qtcloud-devops

## 背景

`code audit` 初次自举时报出大量 `.freezed.dart` 文件超长（500~1900 行），这些是由 Dart `freezed` 库自动生成的代码，过长是常态且不可修复。

## 决策

在 `count_markers` 中维护一个硬编码的 codegen 文件后缀列表，匹配的跳过所有检查（长度、unwrap、TODO 扫描）。

```rust
const GENERATED_SUFFIXES: &[&str] = &[
    ".freezed.dart",  // Dart freezed codegen
    ".g.dart",        // Dart json_serializable
    ".grpc.dart",     // gRPC Dart
    ".pb.dart",       // Protobuf Dart
    ".pb.go",         // Protobuf Go
];
```

## 原则

- 不做配置文件的可配置 exclude 机制（ponytail：够用就停）
- 有新的 codegen 模式直接往数组追加一行
- codegen 文件不应影响门禁判定

## 后果

- 自举超长文件从 15 个降到 9 个（消除 6 个 freezed 噪声）
- 新增 codegen 模式时维护成本极低
