# C. 入口 API + SSE 事件 Schema（初稿）

> 依据：D13（异步提交 + SSE，ADR-0004）、D12（human-input 唤醒）、D24（占位符）、D28。
> 鉴权：`Authorization: Bearer <服务级 API Key / JWT>`（D17，预留 mTLS）；`tenant_id`/`user_id` 取自鉴权主体，控制面可显式覆盖。

## 1. 执行 API

| 方法/路径 | 说明 | 请求体 → 响应 |
|---|---|---|
| `POST /invocations` | 提交执行（异步） | `{workflow_id, version?, inputs, conversation_id?, idempotency_key?, timeout_ms?}` → `202 {invocation_id, status:"pending"}`；幂等键命中 → `200 {invocation_id}`（同一实例，D3） |
| `GET /invocations/{id}` | 状态查询 | → `{status, inputs, outputs, error, node_runs 摘要, waiting_input?}` |
| `POST /invocations/{id}/cancel` | 取消 | → `202`；状态机校验 |
| `POST /invocations/{id}/pause` | 暂停（节点边界生效） | → `202` |
| `POST /invocations/{id}/resume` | 恢复已暂停的 Invocation；重启后的续跑是自动的，不走此端点 | → `202` |
| `POST /invocations/{id}/inputs` | human-input 提交（唤醒） | `{node_id, action, data, resolved_by}` → `202`；仅 `waiting_input` 状态可提交；`action` ∈ user_actions 或 `__timeout` |
| `GET /invocations` | 列表（控制面用） | filters: tenant/user/status/conversation/workflow，分页 |

## 2. 管理 API

| 方法/路径 | 说明 |
|---|---|
| `POST /workflows/import` | 导入 Dify DSL（`.yml` 或 `.ifpkg`）：版本区间 0.1.5–0.7.0 加归一化 → 节点类型校验 → 静态引用校验 → 占位导入 → 产出草稿版本和 import_report；只有「错误」级问题才拒绝（D53、D60） |
| `GET /workflows/{id}/versions`、`GET .../versions/{v}` | 版本查询（含 import_report） |
| `PUT /workflows/{id}/bindings` | 类型级 Node Binding（D31）：`{node_type, target, config}` |
| `PUT /workflows/{id}/versions/{v}/nodes/{node_id}/binding` | 实例级覆盖 |
| `POST /capabilities` / `PUT /capabilities/{id}` / `DELETE ...` | Capability Registry 管理（D9）；密钥两种模式：`secrets_ref`（静态）或 `{{secret.name}}` 占位符 |
| `POST /capabilities/{id}/reload` | 热加载 |

## 3. SSE 事件流

`GET /invocations/{id}/events`
- 响应 `text/event-stream`；支持 `Last-Event-ID` / `?after_seq=` 断线回放（事件已落库，D13）
- 心跳：每 15s 注释行 `: keepalive`
- 事件信封（每条事件）：

```json
{ "invocation_id": "uuid", "seq": 42, "type": "...", "ts": "2026-11-26T10:00:00.123Z", "node_id": "1730000001", "payload": { } }
```

### 事件类型（11 类，D28）

| type | 时机 | payload 要点 |
|---|---|---|
| `invocation_started` | 开始执行 | `{workflow_id, version, inputs 引用}` |
| `node_started` | 节点开始（含 attempt） | `{node_type, attempt}` |
| `node_completed` | 节点成功 | `{outputs 引用/摘要, latency_ms}` |
| `node_failed` | 节点失败（含重试中） | `{error, attempt, will_retry, error_strategy}` |
| `node_skipped` | 分支未选中跳过 | `{reason:"branch_not_taken"}` |
| `llm_delta` | LLM 流式增量 | `{delta, provider, model}` |
| `answer_delta` | answer 节点流式输出 | `{delta}` |
| `waiting_input` | human-input 挂起 | `{actions[], form_schema?, timeout_at}` |
| `input_submitted` | 用户已提交输入 | `{node_id, action}` |
| `invocation_finished` | 终态：成功 | `{outputs}` |
| `invocation_failed` / `invocation_cancelled` | 终态：失败/取消 | `{error}` / `{reason}` |

> 注：`node_skipped` 为实现期补充事件（超出最初 11 类清单），供画布与审计使用；事件枚举保持可扩展。

## 4. 错误模型

```json
{ "code": "INVOCATION_NOT_WAITING", "message": "...", "details": { } }
```
HTTP 语义：400 参数/校验、401/403 鉴权、404 不存在、409 状态冲突（如非 waiting 提交输入）、422 导入出现「错误」级问题，无法成为工作流（附 import_report；「阻断项」与「提示」只进报告、不拒绝，见 grill-record D53）、429 限流（预留）、500 引擎内部错误。

## 5. 契约要点

- 所有响应含 `X-Request-Id`；幂等提交返回同一 invocation_id（D3）
- 外部 HTTP 一律经网关出口（D24）：引擎侧请求携带 `{{secret.*}}` 原样值由网关替换
- 事件顺序：单 invocation 内按 `seq` 严格单调；SSE 回放按 seq 续传
