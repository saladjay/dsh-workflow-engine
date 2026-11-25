# C. 入口 API 与 SSE 事件（修订稿，对应 D1–D63）

> 依据：D12、D13、D17、D35、D36、D37、D45、D46、D48–D56、D59、D60。决策原文见 `docs/grill-record.md`。
> 鉴权：服务级 API Key / JWT，只有控制面持有（D17）；`tenant_id`、`user_id` 由控制面传入。
> 标「待 Round 9」的地方等设计定了再补。

## 1. 执行 API

| 方法/路径 | 说明 |
|---|---|
| `POST /invocations` | 提交执行（异步）。请求 `{workflow_id, version?, inputs, conversation_id?, idempotency_key?, timeout_ms?}`；响应 `202 {invocation_id, status:"pending"}`。`version` 省略时取当前已发布版本。幂等键命中返回 `200 {invocation_id}`，指向同一个 Invocation，不论它的状态。超过准入上限返回 `429` |
| `GET /invocations/{id}` | 状态查询。支持 `?wait=true&timeout=…` 长轮询，到终态或等待输入时返回 |
| `GET /invocations` | 列表。可按 tenant、user、status、conversation、workflow、assignee 筛选 |
| `POST /invocations/{id}/cancel` | 取消，向子工作流传播 |
| `POST /invocations/{id}/pause` | 暂停。节点边界生效：在途节点跑完并落库后停下；暂停期间各类时限停止计时；`waiting_input` 状态下返回 `409` |
| `POST /invocations/{id}/resume` | 恢复已暂停的 Invocation。重启后的续跑是自动的，不走此端点 |
| `POST /invocations/{id}/inputs` | 提交人工输入。请求 `{node_run_id, action, data, resolved_by}`；`action` 取 `user_actions` 里的一个，或 `__timeout` |

提交参数的语义：

- `timeout_ms`：整个工作流的执行时间上限，只累计运行中的时间。等用户输入、暂停、排队期间都不计时。配置优先级是引擎默认值、工作流定义里的值、这里的值。
- `conversation_id`：控制面按「dsh 会话 × 工作流」分配。不传则这次调用没有会话变量，也不产生对话轮次。

## 2. 工作流定义与版本

| 方法/路径 | 说明 |
|---|---|
| `POST /workflows` | 用 Workflow Definition 新建工作流，产生一个草稿版本 |
| `POST /workflows/import` | 导入 Dify DSL（`.yml` 或 `.ifpkg`）：版本区间 0.1.5–0.7.0 加归一化，节点类型校验，静态引用校验，占位导入。产出草稿版本和 `import_report`。可带 `model_mapping` 覆盖规则。只有「错误」级问题才拒绝（`422`） |
| `GET /workflows/{id}`、`GET /workflows/{id}/versions`、`GET /workflows/{id}/versions/{v}` | 查询定义与版本。定义里含起始节点的输入 schema，控制面据此生成工具描述 |
| `PUT /workflows/{id}/versions/{v}` | 更新草稿。已发布的版本不能改，要改就从它派生新草稿 |
| `POST /workflows/{id}/versions/{v}/publish` | 发布。条件是没有阻断项；发布时冻结 Node Binding 并校验它满足节点类型的输入输出契约 |

导入结果的严重度分三级：

| 级别 | 例子 | 处理 |
|---|---|---|
| 错误 | YAML 解析失败、图结构非法、不支持的应用类型（含 Dify 的旧式应用 chat / completion / agent-chat） | 拒绝导入，`422` |
| 阻断项 | 占位节点、引用不可解析、模型引用未映射、知识检索节点没有绑定后端 | 进报告，版本停在草稿，不能发布 |
| 提示 | minijinja 不兼容、插件需要人工处理、文件比 0.7.0 新、secret 类型环境变量已换成占位符 | 进报告，可以发布 |

## 3. Node Binding

| 方法/路径 | 说明 |
|---|---|
| `PUT /node-bindings/defaults` | 类型级默认绑定，请求 `{node_type, target, config}`。只对配置里不带目标地址的节点类型开放：knowledge-retrieval、document-extractor、code、template-transform、human-input |
| `PUT /workflows/{id}/versions/{v}/nodes/{node_id}/binding` | 实例级覆盖 |

改了绑定要重新发布才生效。http-request 和 tool 只允许实例级。

## 4. Capability、Provider 配置、密钥

| 方法/路径 | 说明 |
|---|---|
| `POST /capabilities`、`PUT /capabilities/{id}`、`DELETE /capabilities/{id}`、`POST /capabilities/{id}/reload` | Capability Registry。密钥一律写 `{{secret.name}}` |
| `POST /providers`、`PUT /providers/{id}`、`DELETE /providers/{id}`、`POST /providers/{id}/test` | Provider 配置。`test` 做连通性测试 |
| `POST /secrets`、`PUT /secrets/{id}`、`DELETE /secrets/{id}` | 密钥库。只在 `local` 部署档位存在；`gateway` 档下引擎不持有密钥 |

Capability 和 Provider 配置不随版本冻结：每次调用开始时解析，并给这次 Invocation 留一份快照。Capability 的输入 schema 和节点配置对不上时，调用在开始时就失败。

## 5. 文件

工作流可以产出文件，File 在变量里一律是引用 `{name, mime_type, size, url}`。上传、签名链接、Capability 取回产出文件的方式，待 Round 9。

## 6. SSE 事件流

`GET /invocations/{id}/events`

- 响应 `text/event-stream`；心跳每 15s 一个注释行
- 信封：`{invocation_id, seq, type, ts, node_run_id?, payload}`
- 回放：支持 `Last-Event-ID` 或 `?after_seq=`，只回放已落库的结构性事件

| type | 时机 | 是否落库 | payload 要点 |
|---|---|---|---|
| `invocation_started` | 开始执行 | 是 | `{workflow_id, version}` |
| `node_started` | 节点开始，先落库再执行 | 是 | `{node_type, attempt, parent_node_run_id?}` |
| `node_completed` | 节点成功 | 是 | `{outputs, latency_ms}` |
| `node_failed` | 节点失败 | 是 | `{error, attempt, will_retry, error_strategy}` |
| `node_skipped` | 分支未选中 | 是 | `{reason}` |
| `waiting_input` | 人工输入节点挂起 | 是 | `{actions, form_schema?, assignees?, remind_at?, timeout_at?}` |
| `waiting_input_reminder` | 提醒时限到了 | 是 | `{}` |
| `input_submitted` | 用户已提交输入 | 是 | `{action}` |
| `invocation_paused` / `invocation_resumed` | 暂停 / 恢复 | 是 | `{}` |
| `llm_delta` / `answer_delta` | 流式增量 | 否 | `{delta}`，只在连接存活期间可得 |
| `invocation_finished` / `invocation_failed` / `invocation_cancelled` | 终态 | 是 | `{outputs}` / `{error}` / `{reason}` |

断线重连的客户端从 `node_completed` 或 `invocation_finished` 取完整文本。

## 7. 错误模型

```json
{ "code": "INVOCATION_NOT_WAITING", "message": "...", "details": { } }
```

HTTP 语义：400 参数或校验失败、401/403 鉴权、404 不存在、409 状态冲突（比如不在等待输入时提交输入、在等待输入时暂停）、422 导入出现错误级问题（附 `import_report`）、429 超过准入上限、500 引擎内部错误。

调用级错误码举例：`INVOCATION_TIMEOUT`（执行时间上限到了）、`CAPABILITY_SCHEMA_MISMATCH`、`BINDING_CONTRACT_VIOLATION`（发布时）。

## 8. 契约要点

- 所有响应带 `X-Request-Id`
- 单个 Invocation 内 `seq` 严格单调
- 引擎的所有外部 HTTP 请求都经出口层发出：`gateway` 档把 `{{secret.*}}` 原样转给网关，`local` 档由引擎自己替换并做 SSRF 校验
- 节点至少执行一次：崩溃恢复时在途节点会重跑，HTTP 出站请求带幂等头
