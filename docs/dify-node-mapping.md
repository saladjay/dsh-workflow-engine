# A. Dify 节点映射表（修订稿，对应 D1–D84）

> **版本策略**（D42、D60）：接受 DSL `0.1.5` 至 `0.7.0`，导入期做归一化；比 0.7.0 新的文件照常导入，报告置顶一条警告；低于 0.1.5 的拒绝。接受 `.yml` 和 `.ifpkg`（zip，解包取 DSL）。无论版本号是多少，都按节点 `data.type` 对照本表校验——版本号和内容不可靠地对应。下限不是硬需求：实现期如果 0.1.5 的归一化成本偏高，可以把下限抬到 0.2.0，不需要重新决策。
> **兼容性总原则**：节点 id 原样保留。输出节点的 type 仍是 `end`，源码里没有 `output` 这个类型，`output` 只作防御性别名接受。子图节点（React Flow 的 `parentId` 挂接）归一为引擎的容器节点；容器里每一次节点执行都有自己的 `node_run_id`（D38）。
> **范围图例**：**全量**是完整语义；**基础**是核心语义先行、扩展后置；**占位**是导入为 `unsupported` 占位节点，保留拓扑，发布前必须替换；**阻断**是能导入但发布前必须补配置。
> **绑定点**：该节点类型的实现可以经 Node Binding 切换为自研后端。绑定在发布时冻结（D54，ADR-0014）。表里的「可绑定」都指实例级；只有 knowledge-retrieval、document-extractor、code、template-transform、human-input 五类开放类型级绑定，http-request 和 tool 只允许实例级。
> **待核实**：Dify 在 0.7.0 之后没升版本号又加了 `knowledge-retrieval-v2`、`agent-v2`、human-input v2 等（见 `docs/grill-record.md` 的 F6）。核实之前，本表没有的节点类型一律按占位节点处理。
> **dsh 的画布图**：它不是本表节点的子集。dsh 画布上的工作流进入引擎，要经控制面的转换器（D67，ADR-0015）；映射表还没有出（`docs/open-issues.md` 的 OI-018）。

## 1. 入口 / 出口

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `start` | 用户输入 | ROOT 入口，定义输入字段 schema | 按 start 字段 schema 校验 `inputs`；`sys.files` 等按 File 引用传递 | — | 全量 |
| `end`（界面上叫「输出」） | 输出 | RESPONSE 类；输出变量给调用方；可以为可选 | 收集 outputs，写入 `invocation_finished` 事件并落库 | 可绑定（契约是 outputs map） | 全量 |
| `answer` | 直接回复 | Chatflow 专属，按变量出现顺序流式输出 | 文本模板渲染后以 `answer_delta` 推送（不落库）；回答文本记入这次调用的对话轮次 | 可绑定 | 全量 |
| `trigger-schedule` | 定时触发器 | ROOT 触发入口 | 不进引擎：导入时转成控制面的触发配置 | — | 全量（控制面侧） |
| `trigger-webhook` | Webhook 触发器 | 同上 | 同上 | — | 全量（控制面侧） |
| `trigger-plugin` | 集成触发器 | 同上 | 同上 | — | 全量（控制面侧） |

## 2. LLM / 智能

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `llm` | LLM | prompt_template、context、memory | Provider 主干；流式增量 `llm_delta`；结构化输出走分层管线；每次调用写 `token_usage` | 可绑定 | 全量 |
| `agent` | Agent | Agent 策略节点，内部工具循环 | LLM + 工具循环，工具来自 Capability、MCP、内置 | 可绑定（策略实现） | 基础 |
| `question-classifier` | 问题分类器 | LLM 把输入分到预定义类别 | LLM + 结构化输出，按 class_id 路由 | 可绑定 | 全量 |
| `parameter-extractor` | 参数提取器 | LLM 从文本抽结构化参数 | LLM + 结构化输出，校验失败注入重试 | 可绑定 | 全量 |
| `knowledge-retrieval` | 知识检索 | 从知识库检索 | **阻断**：导入后默认没有后端，发布前必须绑定（类型级或实例级），由 Capability 对接企业知识库。`dataset_ids` 是 Dify 内部的 ID，绑定时按它映射到企业知识库 | 可绑定（类型级或实例级） | 基础 |

LLM 节点的三个要点：

- **模型引用**：DSL 里只写 provider 和模型名。导入时按映射规则解析到 **Provider 配置**：默认按同名匹配，导入请求可以带覆盖规则；解析不了的进导入报告，节点不可运行，直到配上。
- **memory**：Dify 的配置是 `role_prefix`、`window`（enabled、size）、`query_prompt_template`。节点配置了 memory 才生效，不是自动功能。历史由引擎提供：取所属 Conversation 最近若干个对话轮次（D56）。没有 `conversation_id` 的调用没有记忆。
- **结构化输出**：优先用厂商原生的 JSON mode；不支持时降级为提示词约束加 JSON Schema 校验，校验失败把错误注入提示词重试。

知识检索在 dsh 里的现状：节点上选后端，三个后端各带一段响应适配，是实例级绑定的现成先例。这些后端换成 Capability 之后，响应适配放在各后端前面的 HTTP 适配服务里（D71）。

## 3. 逻辑控制

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `if-else` | 条件分支 | 按 cases 走分支 | 条件求值引擎；未选中分支的下游标记 skipped | 可绑定（实例级） | 全量 |
| `variable-aggregator` | 变量聚合器 | 多分支输出汇聚 | 取非 skipped 分支的输出 | — | 全量 |
| `variable-assigner` | （遗留别名） | 旧版的聚合器 | 导入时归一为 `variable-aggregator`，报告注明 | — | 全量（别名） |
| `assigner` | 变量赋值 | 写会话变量、循环变量 | 写 `conversation_variables` 和循环变量；会话变量只有这个节点能写 | — | 全量 |
| `iteration` + `iteration-start` | 迭代 | 对数组每个元素跑子流程 | 容器；每个元素是一次节点执行，内部节点从属于它。`parallel_nums` 默认 10，上限可配（`is_parallel` 是并行开关）；三种 `error_handle_mode` 对齐 | 可绑定（实例级） | 全量 |
| `loop` + `loop-start` / `loop-end` | 循环 | 有状态地反复执行 | 容器；每一轮是一次节点执行；循环变量跨轮保留；三种终止方式对齐 | — | 全量 |

## 4. 数据转换

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `code` | 代码执行 | 沙箱运行 Python / JS | JS 进程内池 + Python 外部 worker 池；worker 可以把生成的文件交回引擎 | 沙箱执行器可绑定 | 基础 |
| `template-transform` | 模板转换 | Jinja2 模板 | minijinja，不兼容的特性进导入报告 | 模板引擎可绑定 | 全量 |
| `document-extractor` | 文档提取器 | 把文件解析成文本 | 内置 txt / md / csv / pdf 文本层；复杂格式经绑定走 Capability | 解析后端可绑定 | 基础 |
| `list-operator` | 列表操作 | 过滤、排序、切片、取元素 | 操作集合对齐 | — | 全量 |

## 5. 集成

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `http-request` | HTTP 请求 | 调外部 API，配置内联 | 原 Dify 语义保留；可以拿到二进制响应并产出文件；SSRF 防护生效 | 只允许实例级绑定 | 全量 |
| `tool` | 工具 | 调插件工具 | `api` 映射 Capability；`mcp` 走远程 MCP 客户端；`builtin` 常用子集按需，其余占位；`workflow` 映射 sub-workflow（Round 12 的默认处理） | 每个工具可单独绑定（实例级） | 分级 |
| `human-input` | 人工介入 | 暂停，等人填表单或审批 | 这一路挂起落库，提交后这一路继续跑；并行的其它分支照常执行（D75，这一点与 Dify 不同，Dify 是整条工作流停住，待核实）。**两级时限**：提醒时限到了发提醒事件，超时时限到了走 `__timeout` 分支。可选的 `assignees`。循环里每一轮的等待各是一条记录 | 可绑定（审批后端） | 全量 |

human-input 的说明：Dify 没有审批人模型，谁拿到表单 token 谁就能提交，`delivery_methods` 的接收人只是投递目标。`assignees` 是本引擎的扩展，权限校验仍在控制面。dsh 画布的 `userinput` 节点经转换器落到这里（D67）。并行分支里可以同时有多处人工输入在等人，每处各是一条等待记录、各是一张卡；引擎把它们都照常发出来，同时只显示哪一张是控制面的显示规则（D84，见 `docs/api-spec.md` 的契约要点）。

## 6. 应用类型

| DSL `app.mode` | 说明 | 处理 |
|---|---|---|
| `workflow`、`advanced-chat` | 工作流、Chatflow | 支持 |
| `chat`、`completion`、`agent-chat` | Dify 的旧式应用，DSL 里是 `model_config` 而不是 `graph` | 错误级，提示先在 Dify 里转换成 Chatflow 再导出 |
| `agent` | 0.7.0 新增的 Agent 应用包 | 不在已定范围内，按错误级处理（待确认） |
| RAG Pipeline（含 `datasource`、`knowledge-index` 节点） | — | 错误级 |

## 7. DSL 里的非节点部分

| 部分 | 处理 |
|---|---|
| `environment_variables` | 属于工作流定义，随版本冻结。导入时值为空的 secret 类型变量换成 `{{secret.变量名}}`，报告注明；非 secret 类型的空值发布前由人补 |
| `conversation_variables` | 引擎持有。会话变量在该 Conversation 第一次被使用时按声明初始化默认值；之后版本新增的变量在第一次被引用时补默认值。`sys.dialogue_count` 取该 Conversation 里已有的调用数（Round 7 的默认处理） |
| `features`（开场白、推荐问题、语音等） | 应用层特性，归控制面，引擎忽略 |
| `dependencies`（插件依赖） | 进导入报告，需要人工处理 |
| 位置、尺寸、选中态等画布字段 | 原样保留，引擎不读 |

## 8. 自有扩展节点（非 Dify 兼容面）

| 节点 type | 说明 | 范围 |
|---|---|---|
| `sub-workflow` | 父 Invocation 经它派生子 Invocation，结果回传；嵌套深度默认上限 5；父节点重跑时不重复派生 | 全量 |
| `http-tool` | 引用 Capability 的节点，与 http-request 并存 | 全量 |
| （dsh 会话变量，没有专用节点） | 画布的 `vars` 节点经转换器变成 `http-tool`，引用指向 dsh 会话变量接口的预置 Capability（部署时写入的种子记录），执行时带上会话引用。变量桶由 dsh 保管，不进引擎（D73） | 全量（经 Capability） |
| （Office 文档，没有专用节点） | Office 文档的生成和编辑经 Capability + `http-tool` 接入：文档服务是独立的 HTTP 服务（D64）。一种操作一个接口，共五种——填充、生成、编辑、把 Markdown 排版成 Word、查询模板的占位符（D76、D82）；模板是引擎文件存储里一类带名字的文件（D77）；默认出错即失败，「允许留空」时缺值填 `{{EMPTY}}`（D80）。格式转换后置（D66）。实现语言待定（`docs/open-issues.md` 的 OI-003）。以后要加专用节点时，它绑定到这些 Capability | 本期（实现语言待定） |

## 9. 节点错误处理（D19）

完全对齐 Dify：节点级配置重试次数和重试间隔，外加异常处理策略。导入时原样保留。

| 配置 | 语义 | 在引擎里的落点 |
|---|---|---|
| 重试（次数、间隔） | 节点失败后按间隔重试；重试算同一次节点执行 | `node_runs.attempt`；事件 `node_failed` 的 `will_retry` |
| `fail-branch` | 重试用尽后走节点上接的异常分支，整个 Invocation 不因此失败 | 事件 `node_failed` 的 `error_strategy`；之后沿异常分支照常调度 |
| `default-value` | 重试用尽后给节点一份默认输出，照常往下走 | 事件 `node_failed` 的 `error_strategy`；之后沿正常出边照常调度 |
| 没有配置异常处理策略 | 重试用尽后 Invocation 失败 | 终态事件 `invocation_failed` |

## 10. 导入流程

1. **解包与解析**：`.ifpkg` 先解包取 DSL。YAML 解析失败是错误。
2. **版本**：低于 0.1.5 是错误；高于 0.7.0 记一条置顶的提示。
3. **归一化**：补默认值、别名映射（`variable-assigner` 到 `variable-aggregator`、参数类型改名）、按内容判断文件在插件化分界之前还是之后。各版本要处理的差异见 `docs/grill-record.md` 的版本时间线。
4. **应用类型**：只留 `workflow` 和 `advanced-chat`，其余是错误。
5. **节点类型校验**：逐个对照本表；不认识的变成占位节点，同时记阻断项。
6. **模型映射**：解析每个 model 引用；解析不了的记阻断项。
7. **环境变量**：secret 类型的空值换成占位符，记提示。
8. **静态引用校验**：`{{#node.var#}}` 和 variable selector 引用不可解析的记阻断项。
9. **节点错误处理**：重试配置和异常处理策略原样保留（D19）。
10. **知识检索**：没有绑定后端的记阻断项。
11. **产出**：一个草稿版本和一份 `import_report`。没有阻断项才能发布。

导入是无损的：原始文件留在 `source_dsl`，引擎里读写的都是同一份定义。dsh 的画布编辑的是画布图，不直接读写它（D67，ADR-0015）。

## 11. 一致性验证（实现期建议）

每个标记为全量的节点类型至少配一对夹具：一份 Dify 导出的 DSL 加期望的执行结果，放进持续集成，用来防止后续改动悄悄破坏兼容。
