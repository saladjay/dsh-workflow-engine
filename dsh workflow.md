# dsh 现有工作流实现核查（F3）

> 核查于 2026-11-26，在有 dsh 源码的机器上做。对象是 dsh 插件仓里名字带 dify 和 workflow 的四个插件，以及它们依赖的内核接口。
> 方法：读源码与文档。四个插件的宿主侧由主会话逐文件读完；设计文档与问题记录、内核接口、周边插件与前端由三个子代理分头核查，它们给出的行号没有逐条复核。
> 证据写成「文件:行号」。插件仓的路径省略前缀 `plugins/`，内核仓的路径省略前缀 `packages/`。
> 本仓库是公开的：本文不写内网地址、账号、凭据，也不写具体业务工作流的名字。
> 结论摘要和对设计的影响见 `docs/grill-record.md` 的「核查结果：dsh 现有工作流实现」。仓库位置见 `HANDOFF.md` 第 8 节。

## 1. 组成

| 包 | 角色 | 入口 |
|---|---|---|
| `dsh-workflow-graph` | 画布图的类型与结构校验。纯库 | `src/types.ts`、`src/validate.ts` |
| `dsh-workflow-canvas` | 画布编辑器，几乎全部在浏览器侧。宿主侧只剩一条健康检查路由 | `src/client/*`；`src/index.ts:28-58` |
| `dsh-dify-importer` | Dify DSL 转画布图。纯库。旧的解析与归一化（IR）还在，但不再用于持久化 | 现行入口 `src/graph-builder.ts`；旧的 `src/parser.ts`、`src/normalizer.ts` |
| `dsh-dify-bridge` | 存档、编译、执行、人工输入的运行管理、HTTP 路由、「导入即工具」，以及几个会话工具 | `src/*` |

画布的存取全部走 bridge 的 `/api/dify-bridge/graph/*`（`dsh-workflow-canvas/src/client/api.ts:8-119`）。

## 2. 持久化格式：画布图

- 形状是 `WorkflowGraph { nodes, edges, vars? }`。节点是 `{ id, kind, title, x, y, data }`，边是 `{ id, source, target, sourceHandle?: 'true' | 'false' }`（`dsh-workflow-graph/src/types.ts:226-240`）。它被声明为唯一持久化格式（同文件 `:2`）。
- 没有子图。迭代体、循环体里的节点和容器节点平铺在同一个 `nodes` 数组里，用普通的边连接（`graph-compiler.ts:444-521`、`graph-builder.ts:316-354`）。
- **表达式是 JavaScript。** 条件、URL、query、body、入参、赋值、循环的条目、输出选择器都是 JS 表达式，可以引用 `args.*`、`R["节点id"]?.字段`，循环里还有 `item`、`index`、`prev`（`nodeDefs.ts:76-236`、`graph-compiler.ts:76-89`）。JS 代码节点直接读写 `R` 和 `args`（`nodeDefs.ts:119-126`）。文本字段用 `{{#id.field#}}`、`{{#sys.x#}}`、`{{item}}`、`{{name}}`（`graph-compiler.ts:91-115`、`:546-556`）。
- 引用在导入和编辑时被改写成「节点标题」，编译期再反解回 id；标题重名的不解析（`graph-compiler.ts:151-156`、`graph-builder.ts:451-455`）。
- 结构校验（`validate.ts:13-105`）：节点种类必须认识；有且只有一个 start；边的两端必须存在；`sourceHandle` 只能是 true 或 false；边结构必须无环（循环不是图上的环）。`unsupported` 节点只告警；`vars` 节点赋值的变量必须先在 `graph.vars` 里定义。
- 图 JSON 里没有格式版本号。

## 3. 节点种类与用量

22 种节点。「快照」是插件仓里 4 份在用工作流的最新版本，共 79 个节点；「开发库」是本机开发环境里的 14 份画布图，只看最新版本。

| kind | 含义 | 配置要点 | 快照 | 开发库 |
|---|---|---|---|---|
| start | 入口，声明输入变量 | `variables[]`，类型 text / paragraph / number / file | 4 | 14 |
| agent | 调一次 LLM | systemPrompt、prompt、provider、model、schema、context | 13 | 61 |
| code | 代码 | `language`（python / javascript）、code、inputs | 30 | 138 |
| ifelse | 二分支 | `cases[].condition` | 6 | 29 |
| answer | 回复文本 | text | 1 | 23 |
| pipeline | 逐元素循环 | items、fromIteration、bodyIds | 4 | 19 |
| knowledge | 知识检索 | datasetIds、query、topK、`version`（选后端） | 3 | 18 |
| tool | 第三方工具 | provider、params、adapterDigest | 3 | 18 |
| userinput | 人工输入 | cardTitle、`fields[]` | 5 | 5 |
| subflow | 子流程 | flow（按名字）、inputs | 2 | 5 |
| end | 输出 | outputName、`outputs[]` | 3 | 5 |
| vars | 写会话变量 | `assigns[]` | 3 | 3 |
| document | 文档解析 | input、`version`（选后端） | 2 | 1 |
| http | HTTP 请求 | url、method、headers、params、body、timeout | 0 | 14 |
| parallel | 每个直接子节点是一个并行分支 | label | 0 | 0 |
| loop | 条件循环 | condition、maxIterations、inputVar、outputVar | 0 | 0 |
| var | 给命名变量赋值 | name、value、operation | 0 | 0 |
| template | `{{name}}` 模板 | template | 0 | 0 |
| vagg | 按入边聚合 | name、mode | 0 | 0 |
| phase | 进度分组 | title | 0 | 0 |
| log | 日志旁白 | message | 0 | 0 |
| unsupported | 导入时不认识的 Dify 节点，不可执行 | originalType、rawJson | 0 | 0 |

- 快照里 30 个 code 节点全是 Python，合计约 127KB，占全部节点配置的七成以上。逻辑的主体在 Python 代码里，不在图结构里。
- `parallel`、`loop`、`var`、`template`、`vagg`、`phase`、`log` 在两处都没有用量。

## 4. 导入：Dify DSL → 画布图（有损）

现行入口是 `dsh-dify-importer/src/graph-builder.ts`。它把 Dify 节点改写成画布节点，不检查 DSL 版本，也不看应用类型（`:445-450`）。

| Dify 节点 | 落点 | 丢失或告警 |
|---|---|---|
| start | start | 其余变量类型按 text 导入并告警 |
| end | end 的 `outputs[]` | — |
| llm | agent | 结构化输出丢弃（schema 置空）；memory 只在没有 user 消息时取 `query_prompt_template`；Dify 原生 provider 置空走默认路由（`:183-216`） |
| code | code | — |
| http-request | http | headers 里的变量引用不支持；authorization 丢弃（`:223-238`） |
| knowledge-retrieval | knowledge，后端选 Dify | — |
| document-extractor | document，后端选默认实现 | — |
| tool（workflow 类型） | subflow | 按名字引用 |
| tool（其余） | provider 在本地注册表里才是 tool，否则 unsupported | — |
| template-transform | javascript 代码节点 `return \`模板\`` | Jinja2 的逻辑全部丢失，只剩 `{{#…#}}`（`:290-291`） |
| answer | answer | — |
| variable-aggregator | vagg（append） | 告警 |
| variable-assigner、assigner | var（set） | 告警；会话变量的语义丢失（`:298-302`） |
| iteration | pipeline 加平铺的体内节点 | 体内有多个入口时只取第一个；体内不支持的节点被丢弃（`:316-354`） |
| if-else（多个 case） | 串联起来的多个二元 ifelse | — |
| iteration-start | 丢弃 | — |
| loop、question-classifier、parameter-extractor、list-operator、agent、human-input、触发器、其余 | unsupported | — |

导入的前置校验要求 DSL 里有 start 和 end 或 answer（`parser.ts:100-107`）。在 dsh 里导入过的 Dify 工作流，原始 DSL 已经无法从画布图还原。

## 5. 存档、发布、多用户

- 数据根目录默认在 dsh 用户目录下，按用户分目录（`dsh-dify-bridge/src/storage.ts:3-6`）：
  - `<用户>/workflows/<workflowId>/meta.json` 和 `versions/<n>.graph.json`，滚动保留最近 10 版（`:15`、`:120-141`）。
  - `<用户>/invocations/invocations.jsonl`，每个用户留 50 条（`:16`、`:294-329`）。
  - `runs/<runId>.json` 是运行记录，里面有图、checkpoint、卡片、事件轨迹和结果（`graph-run-manager.ts:50-104`）。
  - `<用户>/tool-outputs/<调用id>/` 是产物，`relay/` 是结果回执文件。
- `meta.json` 的字段：`latestVersion`、`versions[]`、`format`、`status`（draft / published）、`hidden`（缺省 true）、`scope`（project / public）、`projectId`、`publishedAt`（`storage.ts:19-54`）。
- **保存即生效。** 每次保存都让版本号加一，执行时永远取最新版本（`storage.ts:226-254`、`index.ts:562-571`）。`status` 不参与执行。
- **草稿也会被注册成会话工具。** 注册时只看 `hidden`（`index.ts:539-549`）。
- 「发布」是改 meta 再到项目配置里登记一项资源；「撤回发布」是 status 回到 draft 并注销工具（`deps.ts:293-326`，界面 `PublishDialog.tsx:125-129`）。没有不可变的已发布版本。
- `hidden`、`scope`、`projectId` 是控制面的业务元数据，执行不读它们。
- `shared` 是种子目录。读的时候本人目录没有就回落到 `shared`，写只写本人目录（`storage.ts:96-102`、`:188-197`）。同一个 workflowId 可能在 `shared` 和多个用户目录下各有一份，版本各自分叉。
- 身份取登录账号；未登录返回 401，没有装登录插件时回落到 `shared`（`deps.ts:676-681`、`webserver-adapter.ts:132-136`）。
- 子流程按名字解析：会话路径在「本人目录加 shared」里找（`index.ts:202-207`、`:247-257`），运行管理路径跨全部用户目录找（`:471-493`）。设计文档里「按项目过滤」没有落地。

## 6. 执行

- **三个触发入口，没有定时和 webhook。**
  1. 会话工具：每个非 hidden 的存档注册成一个同名工具，参数由 start 的变量派生，number 映射为 number，其余为 string（`index.ts:539-672`、`workflow-tools.ts:221-237`）。
  2. `POST /graph/trial-run`：画布已经不调用它（`routes.ts:171-196`）。
  3. `POST /graph/runs`：供外部网关使用，见第 11 节。
- **编译。** 画布图被编译成一段 async 函数体，以 `(args, agent, python, knowledge, fetch, document, subflow, tool)` 为自由绑定，另加结果表 `R`、台账 `__results`、进度 `__pg`、变量 `__vars`（`graph-compiler.ts:1-11`、`:779`）。图里有 `unsupported` 节点就拒绝编译（`:159-162`）。
- **运行。** 每一段起一个 worker 线程，在 `vm` 上下文里执行，上下文里没有 require、process、fs；LLM、知识库、文档解析、子流程、工具、变量的读写都经 MessageChannel 回到宿主线程执行（`trial-worker.ts:110-133`、`trial-runner.ts:95-184`）。看门狗到点直接终止线程（`trial-runner.ts:186-190`）。
- **超时。** 段执行的默认值和硬上限都是 900 秒（`trial-runner.ts:35-38`）；会话工具默认 300 秒，上限 900 秒（`workflow-tools.ts:13-15`）；Python 代码节点上限 60 秒（`trial-worker.ts:78`）。
- **并行。** `parallel` 编译成 `Promise.all`（`graph-compiler.ts:435-443`）；`pipeline` 编译成串行的 `for` 循环（`:485-493`）。
- **LLM 节点。** 会话内走宿主的 `subagents.startActivation`，provider 是 `spawn`，parent 是调用方 agent。每个 LLM 节点是一个完整的子 agent，不是一次模型调用（`deps.ts:410-486`）。会话外走宿主的 `llm.stream` 直连（`:509-547`）。两条路都不流式、不统计 token；结构化输出靠提示词加文本解析（`:390-407`）。
- **Python。** 本机解释器起子进程，约定 `main(**args)`，结果写成 `__RESULT__` 加 JSON（`python-runner.ts:124-180`）。网络限制是在 Python 层给 socket 打补丁，默认拒绝私网地址（`:22-93`）。没有文件系统、CPU、内存的限制，用本机已装的包，没有白名单。
- **第三方工具。** 固定形态 `python -m <module>`：参数走 stdin 的 JSON，stdout 只回一行 `{ text, json?, files }`，文件写进指定的输出目录，环境变量走白名单（`tool-registry.ts:1-9`、`:56-67`、`:70-135`）。导入时把注册表条目的指纹钉进节点，运行期复验（`:49-54`、`:146-150`）。
- **子流程。** 按名字递归，嵌套深度上限 8，用调用栈检测环（`index.ts:235-257`）。子流程失败则父流程失败。
- **重启。** 启动时把 `running` 的运行记录一律置为 failed，`awaiting_input` 的不动（`graph-run-manager.ts:137-163`）。没有续跑。失败的运行可以从当前段的 checkpoint 重试（`:377-384`）。
- **单活跃。** 同一会话已有运行中或等待输入的运行时，拒绝再次发起（`graph-run-manager.ts:249-259`、`:302-309`）。进度转发的目标只有一个槽，多会话并行时子流程里的进度标题可能互串（`:121-124`）。
- **结果回给模型。** 完整结果写 relay 文件，路径附在工具结果末尾；进会话的只是紧凑摘要（字符串超 480 截断，数组取前 3 项，深度到 3，总量不超过 3000 字）（`path-relay.ts:16-26`、`workflow-tools.ts:246-280`）。

## 7. 人工输入

- 画布的 `userinput` 节点是分段屏障。段号等于到该节点为止经过的 `userinput` 个数；执行到它时 worker 返回 `{ __suspend__, __R }`，运行记录进入 `awaiting_input` 并落下卡片，等待期间没有 worker（`graph-compiler.ts:119-144`、`:644-675`，`graph-run-manager.ts:424-436`）。提交后把答案并进 `R` 表，新起一个 worker 跑下一段（`:338-362`）。
- 字段：key、label、widget（confirm / select / input / textarea）、options、ref（用上游的值预填）、required、maxLength、helper（`types.ts:154-181`）。没有超时，无限期等待。
- 前端是输入框上方的卡片。每 5 秒轮询一次本会话挂起的运行，运行中每 2 秒轮询运行记录；提交、取消、重试、回放各是一条路由（`dsh-dify-bridge/src/client/RunCardDock.tsx:161-311`，`routes.ts:234-253`）。卡片只渲染第一个字段（`RunCardDock.tsx:316`，子代理）。
- 对会话的告知走宿主的两个接口：`appendEvent` 追加一行不触发模型回合的事件，`declareDeliverables` 直接写交付卡，一次最多 8 个文件，文件必须是宿主机上存在的路径（`workflow-tools.ts:119-202`）。挂起和提交对会话静默。
- **`ask_user_question` 是内核的工具，和待答卡没有代码耦合。** 它由模型调用，调用时停在当前步骤里等答案，只有根 agent 能用，子 agent 调用会报错（`interaction/tool-ask-user`、`interaction/user-questions`，子代理）。工作流内部用不了它，所以 bridge 自己做了待答卡。

| | 内核 `ask_user_question` | bridge 的待答卡 |
|---|---|---|
| 谁触发 | 模型调工具 | 工作流执行到节点 |
| 工具调用 | 挂起等答案 | 立即返回一段文字 |
| 状态在哪 | 宿主的会话投影 | bridge 的运行记录 |
| 答案怎么进 | 恢复工具调用 | 提交后新起一段执行 |

## 8. 会话变量

- dsh-vars 插件提供服务 `get(name, sessionKey?)`、`set(name, value, sessionKey?)`、`list(sessionKey?)`。值只能是字符串；桶按会话键隔离，缺省桶是 `user:shared`，读的时候会话桶没命中就回落到缺省桶（`dsh-vars/src/service.ts:33-115`、`dsh-vars/src/index.ts:80-82`，子代理）。
- 对话模型通过会话工具 `vars_get`、`vars_set` 读写同一个桶（`dsh-vars/src/index.ts:55-108`，子代理）。
- 画布的 `vars` 节点逐条把上游的值求值、转成字符串后写入（`graph-compiler.ts:676-685`）。`graph.vars` 是定义表，编译期严格校验（`:164-200`）。提示词里的 `{{#vars.名#}}` 和点式引用 `vars.名` 编译成运行期读取（`:82-84`、`:104`）。
- 在用的工作流只用它写：把若干结果交给 agent，之后的工具调用再来读，其中有文件的绝对路径，也有整篇文本。
- 它和 Dify 的会话变量不是一个东西：按 dsh 会话隔离、跨工作流共享、agent 可读；不是「会话 × 工作流」。

## 9. 文件

- **输入是路径或 URL 字符串。** `file` 型变量的值就是本机路径或 URL（`types.ts:16-19`）。`paragraph` 型入参的值如果是一个已存在的本机文件（不超过 4MB），执行前会被读成全文（`path-relay.ts:29-56`）。
- **产物落在数据目录下。** Python 和第三方工具把文件写进 `<用户>/tool-outputs/<调用id>/`（`python-artifacts.ts:91-93`）；扫描上限是每次 10 个文件、单文件 50MB（`:38-39`）；启动时清掉不在最近 200 条调用记录里的产物目录（`index.ts:791-805`）。
- **交付靠本机绝对路径。** 文件以 `{ name, path, mime_type? }` 写进工具结果末尾的一个围栏块（`files-meta.ts:8-23`）；下载路由按调用 id 和文件名取文件（`routes.ts:203-216`）；交付卡由宿主按路径读本机文件。
- **Office 文档现在的全部能力是 Markdown 转 docx**，实现是一个 Python 第三方工具（学术模板），既是会话工具，也能在图里当 `tool` 节点使用（`md2doc-tool.ts:17`、`:64-131`）。没有模板填充、编辑既有文件、格式转换。
- 有一个会话工具建立在运行记录之上：按运行 id（或缺省取最近一次完成的运行）读出结果，对报告做条目级修改再重放（`render-report-tool.ts:102-147`）。

## 10. 知识检索与文档解析的后端

两种节点都用「节点上选后端」的办法，后端的地址和凭据在插件配置里，不进节点。

| 节点 | 后端 | 协议 |
|---|---|---|
| knowledge | Dify | Dify 的数据集检索接口；DSL 里的加密 dataset id 经配置映射 |
| knowledge | 一个内部知识库平台 | OpenAPI；鉴权依次是 api-key、用共享密钥现铸的短期 JWT、静态 Bearer；按 IP 直连并改写 Host 头；自签证书可以跳过校验（`codewave-kb.ts:1-30`） |
| knowledge | 一个 MCP 服务 | MCP streamable HTTP，每次检索新建会话（`hpc-kb.ts:1-27`） |
| document | 默认后端 | 先 multipart 上传，再走对话接口的流式返回，单次 120 秒（`document-channels.ts:1-45`） |
| document | 自定义 | 裸 HTTP，`POST {input}` 返回 JSON |

输出统一：knowledge 是 `{ result: [片段…] }`，document 是 `{ text }`（`graph-compiler.ts:593-621`）。这是「换实现不换语义」的实例级切换在现有代码里的先例。

## 11. 外部触发

- dsh-chai-kb-api 插件对外提供 `/api/v1/chai-kb/workflow/run`，是一条 HTTP 反向代理，不是进程内调用：Bearer 令牌全等比较（`dsh-chai-kb-api/src/lib/workflow-runner.ts:56-60`）。
- `POST /run` 转发到 bridge 的 `POST /graph/runs`，返回 jobId；`GET /run/:jobId` 查状态；`POST /run/:jobId/submit` 作答（`dsh-chai-kb-api/src/index.ts:138-233`，子代理）。不同步等待，没有回调，由调用方轮询。
- bridge 的 `POST /graph/runs` 要等第一段执行完才返回（`graph-run-manager.ts:302-324`），而网关等它的上限是 30 秒（子代理）。
- 这条路发起的运行不带会话键，所以待答卡不会出现在任何会话里，只能经网关的 submit 作答（`deps.ts:273-290`）。它按 id 取存档，不看 `hidden`。

## 12. 工作流工具怎样对 agent 可见

现行是四层，没有「会话绑工作流」的运行时绑定（子代理）。

1. bridge 给每个非 hidden 的存档在全局层注册一个工具，工具名由工作流名派生，重名加序号（`index.ts:551-556`、`:670`）。
2. 会话只绑 agent，绑定存在 MySQL 的一张表里（`dsh-custom-agent/src/session-binding.ts:74-129`）。
3. 一个 agent 能用的工具是它选配的工具加上公共工具，数据来自 MySQL 的工具表（`dsh-capability-gate/src/collection.ts:44-60`）。
4. 没选配的工具被收窄掉，提示词里只列能用的工具名（`dsh-capability-gate/src/restrict.ts:44-56`、`:148-184`）。

工作流工具名登记在工具表里才受选配约束；没登记的对所有会话可见，包括已经绑了 agent 的会话。

## 13. 宿主侧事实

来自子代理对内核仓的核查。

- **内核是一份内部分支。** 插件依赖的 `appendEvent`、`declareDeliverables`、工具执行上下文里的 `sessionId`、`dsh-presented` 包，只在这份分支里有，公共 SDK 快照里没有。
- **会话模型里没有用户。** 一个 agent 对应一个会话，`Agent.id` 的类型就是 `SessionId`；会话头里没有用户或所有者字段（`core/session/src/types.ts:21`、`:95-132`，`core/agent/src/types.ts:15-18`）。用户和会话的关系全在插件层：登录插件用 AsyncLocalStorage 保存当前用户，持久化插件按用户分目录。
- **回合与步骤。** 一个步骤是一次模型调用加上它请求的全部工具调用，一个回合是一个或多个步骤（`core/session/src/types.ts:289-386`）。等用户回答时 agent 仍处在回合里。
- **工具调用默认独占。** 只有工具的 `isConcurrencySafe(args)` 严格返回 true 才并行，其余一律独占（`core/tools/src/index.ts:1322-1331`）。工作流工具都没有声明，所以同一会话里的工作流调用是串行的。并行上限默认 10。
- **工具调用的 id 不是可靠的唯一键。** `callId` 由模型提供方签发，流里没有 id 时合成 `call-<序号>`，有的适配器会给空串，内核不去重。进程内每次执行另有一个唯一的 `token`。
- **工具执行上下文拿不到回合号。** `turn` 只有类型声明，运行期没有填充。可用的替代是 `exec.agent.id`。
- **工具返回值必须是无损 JSON。** `NaN`、`undefined`、类实例、循环引用等一律拒绝（`util/values/src/index.ts:72-174`）。
- **工具超时是协作式的。** 超时策略替换 `exec.signal`，不丢弃工具的 promise（`guard/timeout-policy/src/index.ts:55-81`）。
- **子 agent 运行时**的入口是 `startActivation(spec)`；`spawn` 是进程内子 agent 的 provider 名，不带父对话的上下文；活跃子 agent 的上限默认是 8（`subagent/subagent/src/index.ts:195-284`、`subagent/subagent-spawn-in-process/src/index.ts:17-54`）。
- **HTTP 路由只有路径级**，处理器拥有整个响应的生命周期，可以挂 SSE（`host/webserver/src/index.ts:44-53`、`:366-373`）。
- **内核自带一个脚本式的工作流引擎**：用 JavaScript 编排子 agent，原语只有 `agent`、`parallel`、`pipeline`、`phase`、`log`、`args`，在沙箱化的进程里执行（`workflow/workflow-ptc/src/runtime.ts:85-93`、`:232-282`）。它按装配文件随默认 preset 启用，绑了 agent 的会话把它排除掉（`dsh-custom-agent/src/preset.ts:54-58`）。画布早期的 `parallel`、`pipeline`、`phase`、`log` 节点和「编译成脚本」的做法都来自它。
- **模型路由都是 OpenAI 兼容协议**（本机 `settings.yaml`，只核对了协议字段）。
- **dsh 在做会话级的 token 归属，已设计未实施。** 内核新增一个会话投影，把 token 用量记到工具、会话自身、压缩、skill 四类条目上；插件仓新增一个插件，每轮结束时把结果写进 MySQL 的汇总表（插件仓 `docs/superpowers/handoffs/2026-11-23-step-attribution-token-accounting-handoff.md`、`2026-11-25-usage-attribution-store-handoff.md`）。LLM 调用移到引擎之后，这部分用量不会出现在 dsh 的会话账里。
- **dsh 已在用 MySQL**：会话日志的持久化、`dsh-custom-agent` 插件自建的表（子代理）。

## 14. 现有实现的架构性限制

只列会影响新设计的，完整的缺陷清单在插件仓的 `open-issue.md`。

- 隔离的粒度是 dsh 会话：运行记录和变量桶都按会话键分开，没有调用级的隔离。
- 工具调用在会话内串行，子 agent 有全局上限，LLM 节点又是子 agent。并行工作流的效率受这三条约束。
- 执行态只在段边界落盘。进程重启时运行中的工作流一律失败。
- 挂起等人的运行在重启后还活着，但前端没有把它重新呈现出来的路径。
- 子流程按名字解析，改按 workflowId 的强引用一直被延后。
- 一个会话同时只允许一个含人工输入的运行。
- 文件、会话变量、交付卡都假定引擎和 agent 在同一台机器上。

## 15. 没查到或不确定的

- 生产环境里存档的实际数量和状态。本文的用量来自仓库快照和本机开发库。
- 内核自带的工作流引擎在企业环境里是否真的加载成功。启动日志里搜不到它。
- 生产 profile 里 `ask_user_question` 是不是限时模式。仓库里的 preset 都是默认的无限等待。
- 工具表里实际登记了哪些工作流工具名。只有一份种子脚本能确认。
- dsh 和新引擎会不会部署在同一台机器上。这决定文件怎么在两者之间传递。
- 企业出口网关能不能替请求签发需要运行时计算的凭据（见第 10 节的现铸 JWT）。
