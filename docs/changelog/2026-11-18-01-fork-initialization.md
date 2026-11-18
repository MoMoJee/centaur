# Fork 初始化变更记录

## 2026-11-18

- 从上游 `v2026.11.13` 提交 `5041d830dbde1bb66b0a677db6f086defd1eeadd` 建立 `codex/shared-sandbox`，保留上游提交身份与许可证。
- 增加 fork 文档索引、需求、架构、开发、文档维护及运维入口，保留上游 README 和 AGENTS，并接入 fork 工作循环。
- 完成共享沙箱的验收定义、实现顺序、数据库迁移及回退设计。核对上游 API、Codex harness、生命周期、Chart 和 CI 的入口，以这些实现作为后续验证基线。

## 验证状态

暂存文档通过单一 H1、相对链接和锚点、公开内容扫描及 `git diff --cached --check`。核对了标签 SHA、分支、上游指南原文和 LICENSE。

WSL 环境已确认。主代理正在独立执行核心 Rust crate 基线，软件测试结果待记录。共享沙箱功能、候选版本部署和 PVE 升级实测留待后续实施。
