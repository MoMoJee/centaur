# 开发与验证

本地环境为 Ubuntu 24.04.5 WSL2（x86_64），已确认 Node.js `v22.23.2`、Postgres 16、pgvector 与 Rust/Cargo `1.100.0` 可用。以下命令从固定上游标签的源码及 CI 核对。四个核心 Rust crate 的基线由主代理执行，结果待记录。

## 准备环境

- 在 WSL 内运行 Linux 工具链。使用 Rust stable、`rustfmt`、`clippy` 和系统构建依赖。Compose 或 Docker 集群检查前先确认 Docker 服务和容器权限。
- 复现网页 CI 时使用其 Node.js `24.22.2` 和 pnpm `10.34.9`。Vocs 构建另需 Bun。集群层需 Docker、kind、kubectl、Helm、just 和 Bun，kind 以 E2E CI 的 `v0.30.0` 为对照。
- 将相同提交干净克隆到 WSL 的 Linux 原生文件系统，在克隆时设 `core.autocrlf=false`。使用上游 Bash 脚本、Kubernetes E2E 或编译时，从该克隆运行。Windows 检出中已观察到脚本为 CRLF，原生克隆可保留 LF 并避开 DrvFs 的构建 I/O 开销。记录 Git 提交、系统和主要工具版本。

## 分层测试

### Rust 与 Codex 协议

从仓库根执行，对照 [Rust CI](../.github/workflows/ci.yml) 和 [Codex 协议测试](../crates/harness-server-conformance/tests/codex_protocol.rs)。

```bash
cargo fmt --all --check
cargo test --locked -p harness-server
cargo test --locked --manifest-path crates/harness-server-conformance/Cargo.toml --test codex_protocol
cargo clippy --locked --workspace --all-targets -- -D warnings
```

协议 conformance 在独立 Cargo workspace，命令显式指定其 manifest。候选验收包括 A/B/A 的 `threadId` 恢复、迟到事件、interrupt、steer、provider pin，以及 app-server 断线后无重复工具副作用。用 fake app-server 固定通知和断线顺序。

### Postgres 与控制面

按 [DB CI](../.github/workflows/ci.yml) 创建独立临时测试库。CI 的 [启动脚本](../.github/scripts/start-test-postgres.sh) 支持 `paradedb` 和 `pgvector`，只为本地测试启动容器。运行前分别导出指向一次性数据库的 `SESSION_SQLX_TEST_DATABASE_URL` 与 `SESSION_RUNTIME_TEST_DATABASE_URL`，连接参数保存在本地测试环境中。SQLx 迁移用例按 CI 覆盖两种搜索后端。Runtime 使用单独的库并串行执行。

```bash
cargo test --locked -p centaur-session-sqlx
cargo test --locked -p centaur-session-runtime -- --test-threads=1
cargo test --locked -p centaur-api-server
```

核对测试输出及带数据库测试是否跳过。候选额外验证空库和旧快照迁移、重启对账、同键重试、跨对话控制拒绝、事件重放和旧调用数据可读。用 [部署门槛](operations/deployment.md#数据库和回退)演练旧版本恢复。

### 网页与文档

Console 的安装和检查与 [Console CI](../.github/workflows/console-ci.yml) 对齐。测试目标页面若走其他前端，按实际入口补它的回归用例。

```bash
pnpm install --frozen-lockfile --filter @centaur/console-web
cd services/console/web
pnpm typecheck
pnpm lint
pnpm test
pnpm build
```

上游 [Vocs CI](../.github/workflows/docs.yml) 在 `docs` 中执行 `npm ci` 和 `npm run build`。仓库内的 fork Markdown 按[文档维护](documentation.md)另查链接与内容。

### Kubernetes 与浏览器

从仓库根执行。上游 [E2E 脚本](../e2e/stack.sh) 使用专用 kind 集群和独立 kubeconfig，默认从当前工作树构建。需要对照上游已发布镜像时使用 `CENTAUR_E2E_IMAGE_TAG`。记录测试实际使用的镜像与集群。脚本中的 provider 测试使用受控响应，首版候选另外需要经批准的 NewAPI 实测。

```bash
helm lint contrib/chart
bash e2e/stack.sh up
bash e2e/stack.sh test e2e/tests/conversation.test.ts e2e/tests/turns.test.ts e2e/tests/lifecycle.test.ts e2e/tests/custom-provider.test.ts
```

失败时用 `bash e2e/stack.sh logs` 采集诊断，确认专用集群后用 `bash e2e/stack.sh down` 清理。按[需求](requirements.md)在实际网页验证 A/B/A，保存执行关联、事件游标、Pod UID 和进程标记的脱敏证据。Pod 重建单独测试卷内状态和中断提示。WSL 结果满足门槛后才进入 [PVE 升级实测](operations/deployment.md)。
