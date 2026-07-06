# CHANGELOG

## [0.1.0] - 2026-07-06

### Added
- 添加架构决策记录（ADR），包括 devops-code boundary、git CLI 使用、metrics、codegen、offline status 等主题，并保留 ADR-001 和 ADR-002。
- 新增实验报告归档（git-exp / bench-coverage）、DevOps 测试方法论及测试阶段工作纪要。
- 新增 contract.md，描述四维架构实现对照。
- 新增 release modeling report（specification）及更新后的 modeling report（包含 Entry/Record 拆分）。

### Changed
- 重命名文档目录 architecture/ 为 adr/，并调整 git 使用相关文档名称（git-network-use-cli → git-library-strategy → git-library-usage）。
- 更新 git library strategy 文档，补充 gix/git2/CLI 内容。
- 重构 specification 目录为 platform。
- 更新 modeling report，将已完成项移入报告，保留核心观察至 roadmap。

### Removed
- 移除过时的报告（release domain model、code-command fix）。
- 清理多余 ADR，仅保留 ADR-001 和 ADR-002。
