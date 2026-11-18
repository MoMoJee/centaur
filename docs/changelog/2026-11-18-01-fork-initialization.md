# Fork 初始化变更记录

## 2026-11-18

- 从上游 `v2026.11.13` 提交 `5041d830dbde1bb66b0a677db6f086defd1eeadd` 建立 `codex/shared-sandbox`，保留上游提交身份与许可证。
- 增加 fork 文档索引、需求、架构、开发、文档维护及运维入口，保留上游 README 和 AGENTS，并接入 fork 工作循环。
- 完成共享沙箱的验收定义、实现顺序、数据库迁移及回退设计。核对上游 API、Codex harness、生命周期、Chart 和 CI 的入口，以这些实现作为后续验证基线。

## 验证状态

暂存文档通过单一 H1、相对链接和锚点、公开内容扫描及 `git diff --cached --check`。核对了标签 SHA、分支、上游指南原文和 LICENSE。

在 WSL 2 Ubuntu 24.04.5 x86_64 的原生文件系统上验证固定上游提交，使用 Rust/Cargo 1.100.0 和独立的 PostgreSQL 16 + pgvector 0.6.0 临时实例。分别为 Runtime 和 SQLx 配置测试库，测试结束后关闭实例。Cargo 依赖使用锁定版本，预取后离线编译。以下命令均退出 0。

| 检查 | 结果 |
| --- | --- |
| `cargo fmt --all --check` 与 `bash -n services/sandbox/entrypoint.sh` | 通过 |
| `cargo test --offline --locked -p harness-server -p centaur-session-runtime -p centaur-api-server -p centaur-session-sqlx -- --test-threads=1` | 630 通过、0 失败。5 项真实 harness 用例被上游标记为忽略。数据库覆盖 PostgreSQL 路径。 |
| `uv run --no-project --python 3.11 --with tomli-w==1.2.0 python -m unittest discover -s services/sandbox -p 'test_*.py'` | 68 通过 |

2026-11-19 补跑独立的 Codex 协议工作区。`cargo test --offline --locked --manifest-path crates/harness-server-conformance/Cargo.toml --test codex_protocol` 通过 3 项，覆盖请求参数、响应和通知。

通过的是上游组件基线。ParadeDB 专属路径、候选镜像、Kubernetes 和网页端到端、真实模型及生产升级验收属于后续阶段。共享沙箱功能按[实施计划](../plan/2026-11-18-02-shared-sandbox.md)待实施。
