# 共享沙箱架构

本文描述 `v2026.11.13` 的接线及首版拟定契约。实际实现按[实施计划](plan/2026-11-18-02-shared-sandbox.md)验证。

## 上游接线

| 层 | 基线证据 |
| --- | --- |
| API | [路由和授权](../crates/centaur-api-server/src/routes.rs) 使用 `thread_key`，提供消息、执行、interrupt 和 SSE。SSE 已支持 `execution_id` 和 `after_event_id`。 |
| 持久化 | [初始迁移](../crates/centaur-session-sqlx/migrations/0001_session_control_plane.sql) 定义父会话及其消息、执行和事件，并限制同一 `thread_key` 的活动执行。执行请求另有[持久字段](../crates/centaur-session-sqlx/migrations/0052_session_execution_request.sql)。 |
| Runtime | [会话运行时](../crates/centaur-session-runtime/src/lib.rs) 处理去重、输出归属、idle pause 和已知 harness ID 的记录。基线输出解析识别点分隔的 `thread.started`、`run.started` 类型。 |
| Harness | [Codex blocks loop](../crates/harness-server/src/codex.rs) 在进程内维护一个当前线程，提供打开、恢复、steer 和 interrupt 的接线，并按线程和 turn 过滤通知。 |
| Provider | [Chart values](../contrib/chart/values.yaml) 的 `codex.customProviders` 连接 OpenAI 兼容 Responses provider 配置和 iron-proxy 注入。首版拟沿用这条路径固定 NewAPI，浏览器与重试请求沿用实例选定的 provider 和模型策略。 |

## 标识与授权

| 标识 | 作用域与职责 |
| --- | --- |
| `thread_key` | 沿用经鉴权的父会话，持有共享沙箱及其生命周期；解析逻辑对话前先校验主体对父会话的操作权限。 |
| `conversation_id` | opt-in 请求内的可选、不透明逻辑对话 ID。父会话与该 ID 的组合唯一，归属由服务端查证。空值或非法格式按 API 契约拒绝，省略时使用旧路径。 |
| Codex `threadId` | 服务端从已关联的 harness 响应或通知取得，按父会话与逻辑对话持久化。首次打开后原子登记，后续切回时复用该对话的 ID。网页输入无权直接指定它。 |
| `execution_id` | 每次执行的持久身份，绑定父会话、逻辑对话和本次沙箱代次。执行期间再关联 Codex `threadId`、`turnId` 与输出所有权。 |

省略 `conversation_id` 的流量沿用旧数据和路由语义。共享模式的启用由实例授权和父会话策略控制。已有父会话默认沿用原生命周期；首版优先使用新父会话启用共享模式，避免把原单线程状态隐式归入新对话。共享对话的历史与事件也须在旧读取路径过滤，避免同父会话的默认调用混收。

## 执行、事件与恢复

1. 在持久层用父会话和逻辑对话限定幂等键，匹配输入指纹后复用原执行。启动前取得同沙箱串行执行权。竞争执行返回可重试的忙响应；排斥范围包含默认调用。
2. 持久执行归属后传递可信执行上下文。首轮从 app-server 的线程创建响应或 `thread/started` 记录真实 ID，随后按该逻辑对话恢复或选用已加载的线程。已记录线程恢复被拒时返回可诊断的失败，保留原映射供恢复或显式重建。
3. SSE、消息及最终输出按授权父会话、逻辑对话和执行过滤。SSE 使用既有游标机制并在回放和实时订阅阶段应用相同过滤。页面切换只改变展示目标，已启动执行有独立终态。
4. 重复请求、网络重试和进程恢复都以原执行记录为准。若派发确认未知，先对账已持久状态及当前所有权；无法确认时关闭为待核查失败，保留原执行，避免静默重放可能已经产生副作用的 turn。
5. interrupt、steer 经授权校验并匹配当前活动执行。共享模式下若目标已终止，返回过期控制状态，后续执行仍需新请求。匹配后由 harness 验证 `threadId` 和 `turnId`。异步通知使用发出时的执行上下文归属。迟到或无法关联的通知进入隔离诊断，不能更新另一条执行、终态或对话映射。终态、超时和断连恢复通过执行所有权条件更新，避免旧拥有者覆盖新执行。

## 运行生命周期

共享父会话拟持久化 `keep_running` 策略，默认关闭，由实例显式启用。启用期间跳过本沙箱的自动 idle pause 和最大年龄回收。pause 与 reaper 在动作前复核持久策略和沙箱代次。单轮执行仍有独立超时，触发后终止并记录该轮。显式停止、Pod 故障和节点维护保有各自路径。状态缺失或集群无法确认原 Pod 时，向页面报告沙箱中断，重新确认沙箱代次再接收新执行。

同一运行 Pod 内可以保留容器可写层、文件和进程。启用 `sandbox.stateVolume` 的新 agent-k8s 沙箱可让落在该卷上的工作区与 Codex 状态跨 Pod 重建。进程、内存和卷外临装软件随原容器消失。检查 PVC 实际挂载、卷内路径和 Codex 恢复结果。最终删除沙箱会触发清理 PVC，备份与恢复验证见[部署验收](operations/deployment.md)。
