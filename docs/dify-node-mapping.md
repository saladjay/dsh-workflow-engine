# A. Dify 节点完整映射表（DSL 0.7.0 基线）

> 基线：Dify DSL `version: "0.7.0"`（Dify 1.16+，稳定版 1.20.0）。承诺**导入执行**，不承诺反向导出（ADR-0002）。
> 兼容性总原则：节点 id 原样保留；输出节点的 type 仍是 `end`（源码复核：不存在 `output` 类型，`output` 仅作防御性别名接受）；子图节点（React Flow `parentId` 挂接）归一为引擎 IR 的容器节点。
> 范围图例：**全量**=完整语义实现｜**基础**=核心语义先行、扩展后置｜**占位**=导入为 `unsupported` 占位节点（保留拓扑、执行报错，D14）。
> 绑定点：该节点类型的实现可通过 **Node Binding**（D31，ADR-0009）切换为自研后端。

## 1. 入口 / 出口

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `start` | 用户输入 | ROOT 入口，定义输入字段 schema | API 提交的 `inputs` 按 start 字段 schema 校验；`sys.files` 等大对象引用传递（D15） | — | 全量 |
| `end`（界面上叫「输出」） | 输出 | RESPONSE 类；输出变量给调用方；现可为可选节点 | 收集 outputs → `invocation_finished` 事件 + 持久化；源码里没有 `output` 这个 type，`output` 仅作防御性别名接受 | 可绑定（契约 = outputs map） | 全量 |
| `answer` | 直接回复 | RESPONSE 类；Chatflow 专属，按变量出现顺序流式推送 | 文本模板渲染（含 `{{#node.var#}}`）→ `answer_delta` 增量事件；多处 answer 节点按到达顺序流式 | 可绑定 | 全量 |
| `trigger-schedule` | 定时触发器 | ROOT 触发入口 | **不进引擎**：导入时转换为控制面（dsh）触发配置（ADR-0006）；引擎只收 API 触发 | — | 全量（控制面侧） |
| `trigger-webhook` | Webhook 触发器 | ROOT，HTTP 回调触发 | 同上 | — | 全量（控制面侧） |
| `trigger-plugin` | 集成触发器 | ROOT，插件事件触发 | 同上 | — | 全量（控制面侧） |

## 2. LLM / 智能

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `llm` | LLM | prompt_template + context + memory | Provider 主干（D5：OpenAI 兼容 + 适配器）；流式 → `llm_delta`；结构化输出走 D6 分层；memory = 从 **Conversation** 注入历史轮次；每次调用写 token_usage（D18） | 可绑定 | 全量 |
| `agent` | Agent | Agent 策略节点，内部工具循环 | 基础版：LLM + 工具循环，工具来自 Capability/MCP/builtin；策略对齐 function-calling 优先（D27），ReACT 等模板后置 | 可绑定（策略实现） | 基础 |
| `question-classifier` | 问题分类器 | LLM 把输入分类到预定义类别 → 分支 | LLM + D6 结构化输出（类别枚举）→ 按 class_id 路由分支 | 可绑定 | 全量 |
| `parameter-extractor` | 参数提取器 | LLM 从文本抽结构化参数 | LLM + D6（json_schema = 提取参数定义）；校验失败注入重试 | 可绑定 | 全量 |
| `knowledge-retrieval` | 知识检索 | 从知识库检索相关内容 | 默认实现 = Capability（企业知识库 HTTP API，D22/D25）；多知识库 = 多 Capability；输入输出契约固定（query/top_k → docs） | **绑定示范点** | 基础 |

## 3. 逻辑控制

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `if-else` | 条件分支 | 按 cases 条件走分支；sourceHandle = case_id 或 `false` | 条件求值引擎（操作符集合对齐 Dify）；未选中分支下游标记 skipped（D29） | 条件表达式引擎可绑定 | 全量 |
| `variable-aggregator` | 变量聚合器 | 多分支输出汇聚到同一变量 | 取非 skipped 分支的首个非空输出（对齐 Dify） | — | 全量 |
| `variable-assigner` | （遗留别名） | 旧版聚合器别名 | **导入时归一为 `variable-aggregator`**（兼容处理，报告注明） | — | 全量（别名） |
| `assigner` | 变量赋值 | 写入会话变量/循环变量（Variable Assigner v2） | 写 conversation_variables / loop_variables；会话变量仅此节点可写（D20，对齐 Dify） | — | 全量 |
| `iteration` + `iteration-start` | 迭代（子图） | 对数组每元素跑子流程，输出数组；`item`/`index` 上下文 | 容器子图；`is_parallel` 默认 10 保持兼容，**上限解锁为配置值**（D29）；error_handle_mode 三种对齐：terminated / continue-on-error（失败位填 null）/ remove-abnormal-output | 执行策略可配置 | 全量 |
| `loop` + `loop-start` / `loop-end` | 循环（子图） | 有状态反复执行子流程直到退出 | 循环变量跨轮持久（assigner 更新）；三种终止对齐：break_conditions / loop_count / loop-end；loop_count 必填防死循环 | — | 全量 |

## 4. 数据转换

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `code` | 代码执行 | 沙箱运行 Python/JS（`def main(...)`） | JS = rquickjs 进程内池；Python = 外部 worker 池（D30：python-build-standalone + uv 预装 + 白名单 + seccomp/cgroups，容器池=安全档位）；worker 预置 main() shim，出入参 JSON | 沙箱执行器可绑定（如换容器执行器） | 基础 |
| `template-transform` | 模板转换 | Jinja2 模板拼接字符串 | minijinja（D27：高兼容非 100%，边界特性降级并在导入报告记录） | 模板引擎可绑定 | 全量 |
| `document-extractor` | 文档提取器 | 解析文件变量为文本 | 内置 txt/md/csv/pdf 文本层；复杂格式（docx/pptx/OCR）→ Capability 文档解析服务（D27）；文件引用传递（D15） | 解析后端可绑定 | 基础 |
| `list-operator` | 列表操作 | 过滤/排序/切片/取元素 | 操作集合对齐 Dify | — | 全量 |

## 5. 集成

| DSL type | 中文 | 执行语义 | 本引擎实现 | 绑定点 | 范围 |
|---|---|---|---|---|---|
| `http-request` | HTTP 请求 | 调外部 API（节点内联配置） | 原 Dify 语义保留（D26）；占位符透传（D24）；SSRF 防护生效 | 可绑定 → http-tool | 全量 |
| `tool` | 工具 | 调插件工具（provider_type: builtin/api/workflow/mcp） | 分流：`api`→**Capability Registry**（全量）；`mcp`→MCP 客户端（基础）；`builtin`→常用子集按需、其余占位；`workflow`→**映射 sub-workflow**（调用另一工作流，全量）；每个工具可单独绑定 | 每工具可绑定 | 分级 |
| `human-input` | 人工介入（HITL） | 暂停等人填表单/审批后继续；sourceHandle = user_actions[].id 或 `__timeout` | D12 全套：异步事件唤醒 + 控制面长轮询封装、单节点模式字段（审批/表单/混合）、挂起快照落库重启可唤醒、超时走 timeout 分支 | 可绑定（审批后端 = 企业审批系统 Capability） | 全量 |

## 6. RAG Pipeline 专用（本期不支持该应用类型）

| DSL type | 中文 | 说明 | 范围 |
|---|---|---|---|
| `datasource` | 数据源 | RAG Pipeline 专用入口 | 导入报错提示"RAG Pipeline 应用类型暂不支持" |
| `knowledge-index` | 知识库 | RAG Pipeline 专用索引节点 | 同上 |

## 7. 自有扩展节点（非 Dify 兼容面）

| 节点 type | 说明 | 绑定点 | 范围 |
|---|---|---|---|
| `sub-workflow` | 父 Invocation 经此派生子 Invocation，结果回传父级；嵌套深度默认上限 5（D11/D29）；Dify `tool(workflow)` 映射到此 | 目标可绑定（外部引擎实例） | 全量 |
| `http-tool` | 引用 **Capability** 的唯一节点类型（D9/D26）；与 http-request 并存 | capability 可换 | 全量 |

## 8. 导入行为汇总

1. 版本检测：接受 `0.1.5` 至 `0.7.0`，旧文件先做导入期归一化再按 0.7.0 校验；比 0.7.0 新的文件照常导入并在报告置顶警告，不拒绝；低于 0.1.5 的拒绝。无论版本号是多少，都按节点 `data.type` 对照引擎支持的类型校验。导入接口一并接受 `.ifpkg`（解包取 DSL，资源忽略）。（D42、D60，见 grill-record Round 5 与 Round 8）
2. 逐节点映射：上表命中 → 正常导入；未命中 → `unsupported` 占位节点（保留拓扑，导入报告列出，执行到该节点报明确错误）
3. 静态校验（D15）：所有 `{{#node.var#}}` 与 variable selector 引用可解析（节点存在、路径存在），失败项进导入报告
4. `dependencies`（插件依赖）：`api` 类映射 Capability 提示；marketplace 插件类在导入报告标注需人工处理
5. 产出：导入报告（节点清单、unsupported 列表、校验错误、绑定建议）+ 内部 IR（graph JSON）
