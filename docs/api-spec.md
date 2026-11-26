# C. 入口 API 与 SSE 事件（修订稿，对应 D1–D75）

> 依据：D3、D11–D13、D17–D19、D35–D38、D40、D43、D45–D50、D53–D56、D59–D61、D64–D75，以及各轮的默认处理。决策原文见 `docs/grill-record.md`。
> 鉴权：服务级 API Key / JWT，只有控制面持有（D17）。`user_id` 由控制面传入；`tenant_id` 是部署级的固定值，由引擎读环境变量，控制面不用传（D68）。
> 还没定的：标了 OI 编号的接口细节。

## 1. 执行 API

| 方法/路径 | 说明 |
|---|---|
| `POST /invocations` | 提交执行（异步）。请求 `{workflow_id, version?, user_id, inputs, conversation_id?, session_ref?, idempotency_key?, timeout_ms?}`；响应 `202 {invocation_id, status:"pending"}`。`version` 省略时取当前已发布的版本；带了 `version` 时，它必须是当前已发布的版本，否则返回 `409`（`WORKFLOW_VERSION_UNPUBLISHED`）；工作流没有已发布的版本时也是 `409`（D74）。幂等键先于版本校验：命中时返回 `200 {invocation_id}`，指向同一个 Invocation，不论它的状态，也不再校验版本。超过准入上限返回 `429` |
| `GET /invocations/{id}` | 状态查询。支持 `?wait=true&timeout=…` 长轮询，到终态、进入暂停，或者出现新的未完成等待记录时返回。调用状态里的「等待输入」指没有任何节点在执行、只在等人（D75）；一路在等人、另一路在跑时，状态是运行中，但等待记录已经存在，长轮询照样返回 |
| `GET /invocations` | 列表。可按 tenant、user、status、has_waiting_input（有未完成的等待记录，不看调用状态）、conversation、workflow、session_ref、assignee 筛选，可按创建时间排序（用来取某个会话、某个工作流最近一次成功的调用） |
| `GET /invocations/{id}/node-runs` | 这次调用的节点执行记录，含每个节点的输入输出。控制面用它做事后排查，以及「对已完成的运行的报告做修改」这类工具（Round 11 的默认处理） |
| `POST /invocations/{id}/cancel` | 取消，向子工作流传播 |
| `POST /invocations/{id}/pause` | 暂停。节点边界生效：在途节点跑完并落库后停下；暂停期间各类时限停止计时；`waiting_input` 状态下返回 `409`。暂停期间提交的输入照常记下，恢复之后才往下调度（Round 12 的默认处理） |
| `POST /invocations/{id}/resume` | 恢复已暂停的 Invocation。重启后的续跑是自动的，不走此端点 |
| `POST /invocations/{id}/inputs` | 提交人工输入。请求 `{node_run_id, action, data, resolved_by}`；`action` 取 `user_actions` 里的一个。`__timeout` 只由引擎在超时时限到了的时候触发，这个端点不接受它（Round 12 的默认处理）。要提交的那个节点没有处于等待的记录时返回 `409`。判断看等待记录，不看调用状态：一路在等人、另一路在跑时，调用是运行中，照样可以提交 |

提交参数的语义：

- `user_id`：登录账号的 id，必填，由控制面传入（D68）。
- `timeout_ms`：整个工作流的执行时间上限，只累计运行中的时间，按调用状态计：状态是 `running` 时才计时，调用整体在等用户输入、暂停、排队期间不计时。一路在等人、另一路在跑时，调用是运行中，照常计时。配置优先级是引擎默认值、工作流定义里的值、这里的值。到上限时 Invocation 判失败（`INVOCATION_TIMEOUT`），在途的节点和子工作流一起取消，没有分支可走（Round 7 的默认处理）。子 Invocation 的截止时间不超过父 Invocation 的剩余时间（Round 5 的默认处理）。
- `conversation_id`：控制面按「dsh 会话 × 工作流」分配。不传则这次调用没有会话变量，也不产生对话轮次。
- `session_ref`：控制面会话的不透明引用，引擎不解析，只用来筛选，以及让控制面在重启或重连之后找回有未完成等待记录的调用（Round 10 的默认处理；按等待记录找，用列表接口的 `has_waiting_input`，不按调用状态）。dsh 会话变量的节点也用它来寻址（D73），所以它要在变量池里能被节点引用。
- 默认的执行时间上限是 1 小时，可以配置（Round 10 的默认处理）。

## 2. 工作流定义与版本

| 方法/路径 | 说明 |
|---|---|
| `POST /workflows` | 用 Workflow Definition 新建工作流，产生一个草稿版本 |
| `POST /workflows/{id}/versions` | 给已有的工作流新建一个草稿版本。从发布过的版本（已发布或已下线）派生，不带参数时取当前已发布的那个；没有已发布的版本时必须指定来源。一个工作流同一时刻只有一个草稿版本，已经有草稿时返回 `409`（D59；Round 7 的默认处理「要改就从已发布版本派生新草稿」） |
| `POST /workflows/import` | 导入 Dify DSL（`.yml` 或 `.ifpkg`）：版本区间 0.1.5–0.7.0 加归一化，节点类型校验，静态引用校验，占位导入。产出草稿版本和 `import_report`。可带 `model_mapping` 覆盖规则。不带 `workflow_id` 时新建工作流；带上时在该工作流下新建一个草稿版本，该工作流已经有草稿时返回 `409`。只有「错误」级问题才拒绝（`422`） |
| `GET /workflows/{id}`、`GET /workflows/{id}/versions`、`GET /workflows/{id}/versions/{v}` | 查询定义与版本。定义里含起始节点的输入 schema，控制面据此生成工具描述 |
| `PUT /workflows/{id}/versions/{v}` | 更新草稿。发布过的版本（已发布、已下线）不能改，要改就从它派生新草稿 |
| `POST /workflows/{id}/versions/{v}/publish` | 发布。条件是没有阻断项；发布时冻结 Node Binding 并校验它满足节点类型的输入输出契约。发布新版本时，之前已发布的那个自动下线（D74）。重新发布一个下线过的版本（回滚）时，沿用当时冻结的绑定，并且仍然校验这些绑定目标存在、契约满足；校验不过就拒绝发布，当前已发布的版本不变 |
| `POST /workflows/{id}/versions/{v}/unpublish` | 下线。只对当前已发布的版本有效。下线后这个工作流没有已发布的版本，不再接受新的调用；在途的 Invocation 照常用它开始时的版本跑完；之后可以重新发布（D69、D74）。控制面的「撤回发布」在调用它之外还要注销工具 |

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
| `PUT /workflows/{id}/nodes/{node_id}/binding` | 实例级覆盖，作用于当前的草稿版本（一个工作流同一时刻只有一个草稿版本）。没有草稿时返回 `409` |

改了绑定要重新发布才生效。http-request 和 tool 只允许实例级。

## 4. Capability、Provider 配置、密钥

| 方法/路径 | 说明 |
|---|---|
| `POST /capabilities`、`PUT /capabilities/{id}`、`DELETE /capabilities/{id}`、`POST /capabilities/{id}/reload` | Capability Registry。密钥一律写 `{{secret.name}}` |
| `POST /providers`、`PUT /providers/{id}`、`DELETE /providers/{id}`、`POST /providers/{id}/test` | Provider 配置。`test` 做连通性测试 |
| `POST /secrets`、`PUT /secrets/{id}`、`DELETE /secrets/{id}` | 密钥库。只在 `local` 部署档位存在；`gateway` 档下引擎不持有密钥 |

Capability 和 Provider 配置不随版本冻结：每次调用开始时解析，并给这次 Invocation 留一份快照。Capability 的输入 schema 和节点配置对不上时，调用在开始时就失败。

Capability 的定义里可以指定文件参数的传递方式（内联或预签名地址，见第 5 节）。Office 文档服务是一个独立的 HTTP 服务，接口注册成 Capability，工作流里用 http-tool 节点引用（D64）。

后端的响应和节点的输出契约不一致时，在后端前面放一个 HTTP 适配服务，Capability 指向适配服务；Capability 的定义里不做响应变换（D71）。

指向 dsh 会话变量读写接口的 Capability 是部署时写进去的一组预置记录，和别的 Capability 一样可以改、可以删，不是代码里的特例（Round 12 的默认处理）。画布的 `vars` 节点经转换器变成引用它们的 http-tool 节点（D73）。dsh 一侧要提供带服务令牌的接口（`docs/open-issues.md` 的 OI-063）。

## 5. 文件

工作流可以产出文件，File 在变量里一律是引用 `{name, mime_type, size, url}`。

文件在一次 HTTP 调用里怎么进出（D65）：

- 默认自动。小于可配阈值（默认 10MB）的文件内联在 JSON 里；超过的用预签名地址：引擎向文件存储申请有时效的下载和上传地址，服务用 HTTP 取输入、回传输出。
- Capability 可以在定义里指定只用其中一种。
- 本地目录的开发档位下，由引擎自己提供带令牌的上传和下载端点。

dsh 与引擎之间由控制面搬运（Round 10 的默认处理）：调用前把 file 型入参里的本机文件上传到引擎的文件存储；调用结束后把产出文件下载到 dsh 本机目录，再声明交付卡。

上传、下载、签名链接的具体接口在实现期定（`docs/open-issues.md` 的 OI-021）。

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
| `input_submitted` | 用户已提交输入；等待超时时由引擎触发，`action` 是 `__timeout`、`resolved_by` 是 `engine` | 是 | `{action, resolved_by}` |
| `node_cancelled` | 节点被取消：Invocation 被取消或到了执行时间上限时，在途的节点和挂在等待上的节点 | 是 | `{reason}` |
| `invocation_paused` / `invocation_resumed` | 暂停 / 恢复 | 是 | `{}` |
| `llm_delta` / `answer_delta` | 流式增量 | 否 | `{delta}`，只在连接存活期间可得 |
| `invocation_finished` / `invocation_failed` / `invocation_cancelled` | 终态 | 是 | `{outputs}` / `{error}` / `{reason}` |

断线重连的客户端从 `node_completed` 或 `invocation_finished` 取完整文本。

## 7. 错误模型

```json
{ "code": "WAITING_INPUT_NOT_FOUND", "message": "...", "details": { } }
```

HTTP 语义：400 参数或校验失败、401/403 鉴权、404 不存在、409 状态冲突（比如向一个没有在等待的节点提交输入、在调用处于等待输入时暂停、已有草稿时再新建草稿、没有草稿时改绑定、带着的版本不是当前已发布的版本）、422 导入出现错误级问题（附 `import_report`）、429 超过准入上限、500 引擎内部错误。

调用级错误码举例：`INVOCATION_TIMEOUT`（执行时间上限到了）、`CAPABILITY_SCHEMA_MISMATCH`、`BINDING_CONTRACT_VIOLATION`（发布时）、`WORKFLOW_VERSION_UNPUBLISHED`（带着的版本不是当前已发布的那个——是草稿或已下线；或者工作流没有已发布的版本）。

## 8. 契约要点

- 所有响应带 `X-Request-Id`
- 一个 dsh 会话里同时只有一个带未完成等待记录的调用。这是控制面自己的限制（D72），按等待记录判断，不按调用状态；引擎不强制
- 一个工作流同一时刻只有一个已发布的版本。发布新版本时旧版本自动下线；在途的 Invocation 用它开始时的版本跑完；带着版本号调用只认当前已发布的版本；回滚就是把旧版本重新发布，沿用当时冻结的绑定（D74）
- Invocation 的状态：只要有节点在执行就是 `running`；没有节点在执行、只剩等人输入时才是 `waiting_input`。没被等待挡住的分支照常推进；多入边的节点要等所有入边所在的分支都完成才调度，包括其中等人输入的那一支；被跳过的分支算完成（D75）
- 单个 Invocation 内 `seq` 严格单调：序号来自同一个计数器，流式增量也占号；重启后从库里该 Invocation 的最大序号续（Round 12 的默认处理）
- 引擎的所有外部 HTTP 请求都经出口层发出：`gateway` 档把 `{{secret.*}}` 原样转给网关，`local` 档由引擎自己替换并做 SSRF 校验
- 需要运行时计算的鉴权（现铸 JWT 那类）不进出口层的占位符机制，这类后端包成 HTTP 服务再注册为 Capability（Round 10 的默认处理第 7 条）
- 节点至少执行一次：崩溃恢复时在途节点会重跑，HTTP 出站请求带幂等头
- 日志只输出结构化日志（带 `invocation_id`、`node_run_id`），引擎不存日志；追踪用 OpenTelemetry，可选（Round 5 的默认处理）
