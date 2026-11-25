# Dify Workflow/Chatflow 系统调研报告(2025-2026 现状)

## 0. 版本背景(影响所有结论)

- 当前稳定版为 Dify 1.x(最新 tag `1.20.0`),`2.0.0-beta.1/.2` 已打 tag;`main` 分支已把图执行引擎抽到独立库 **`langgenius/graphon`**(pyproject pin `graphon==0.8.0` + git rev)。
- **节点 type 字符串在 1.9.0 → main/graphon 之间完全一致**，只是定义位置从 `api/core/workflow/enums.py` 的 `NodeType(StrEnum)` 迁移到 `graphon/src/graphon/enums.py` 的 `BuiltinNodeTypes`(不再是 enum,node_type 就是普通字符串)。新增了 `human-input` 和 3 个 trigger 类型。
- DSL 版本:`api/constants/dsl_version.py` 中 `CURRENT_APP_DSL_VERSION = "0.7.0"`(1.15.x 及更早为 `"0.6.0"`,1.16 起 `"0.7.0"`,仅凭版本号无法区分 1.16/1.18)。
- **End 节点已更名为 Output(“输出”)**，且从“每个 workflow 必需”变为“可选”；Chatflow 仍必须以 Answer 节点收尾。
- Loop(循环)节点在 v1.2.0 引入，替代了旧版“用迭代模拟循环”的做法。

---

## 1. 完整节点类型清单

权威来源:`graphon/src/graphon/enums.py` 的 `BuiltinNodeTypes` + `api/constants/trigger.py` + 交叉验证 `1.9.0/api/core/workflow/enums.py`、前端 `web/i18n/locales/zh-Hans/workflow.json` 的 `blocks.*`。

| DSL `data.type` 值    | 中文名(官方 zh-Hans) | 一句话语义                                                   | 子图/分支语义                                              |
| --------------------- | -------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| `start`               | 用户输入             | 工作流入口，定义用户输入字段(ROOT)                           | 无                                                         |
| `end`                 | 输出(旧名 End)       | 输出变量给调用方/API,workflow 专属，现在可选                 | 响应节点(RESPONSE)                                         |
| `answer`              | 直接回复             | 流式输出文本/变量/图片给用户，Chatflow 专属                  | 响应节点(RESPONSE),可多个、按变量顺序流式                  |
| `llm`                 | LLM                  | 调用大模型(prompt_template + context + memory)               | 无                                                         |
| `knowledge-retrieval` | 知识检索             | 从知识库检索相关内容                                         | 无                                                         |
| `if-else`             | 条件分支             | 按条件走不同分支                                             | **分支**(BRANCH);sourceHandle = case id 或 `false`         |
| `question-classifier` | 问题分类器           | LLM 把输入分类到预定义类别                                   | **分支**(BRANCH);sourceHandle = class id                   |
| `code`                | 代码执行             | 沙箱运行 Python/JS(`def main(...)`)                          | 无                                                         |
| `template-transform`  | 模板转换             | Jinja2 模板拼接字符串                                        | 无                                                         |
| `http-request`        | HTTP 请求            | 调外部 API(SSRF 代理)                                        | 无                                                         |
| `tool`                | 工具                 | 调插件工具(provider_type: builtin/api/workflow/mcp)          | 无                                                         |
| `agent`               | Agent                | Agent 策略节点(含 v2,Dify Agent runtime)                     | 无(内部可有工具循环)                                       |
| `parameter-extractor` | 参数提取器           | LLM 从文本抽结构化参数                                       | 无                                                         |
| `document-extractor`  | 文档提取器           | 解析文件变量为文本                                           | 无                                                         |
| `list-operator`       | 列表操作             | 对列表做过滤/排序/切片/取元素                                | 无                                                         |
| `variable-aggregator` | 变量聚合器           | 多分支输出汇聚到同一变量                                     | 聚合(多入一出来汇合分支)                                   |
| `variable-assigner`   | 变量赋值器           | **遗留值**：旧版聚合器别名(= variable-aggregator 的兼容 type,勿与新赋值器混淆) | 兼容别名                                                   |
| `assigner`            | 变量赋值             | 写入会话变量/循环变量(Variable Assigner v2)                  | 无                                                         |
| `iteration`           | 迭代                 | 对数组每个元素跑子流程，输出数组                             | **容器**(CONTAINER 子图)，含 `iteration-start` 子节点      |
| `iteration-start`     | 迭代开始             | 迭代子图的伪起始节点                                         | 子图入口                                                   |
| `loop`                | 循环                 | 有状态反复执行子流程直到退出条件                             | **容器**(CONTAINER 子图)，含 `loop-start`                  |
| `loop-start`          | 循环开始             | 循环子图入口                                                 | 子图入口                                                   |
| `loop-end`            | 退出循环(Exit Loop)  | 立即终止循环                                                 | 子图出口控制                                               |
| `human-input`         | 人工介入(HITL)       | 暂停工作流，等人填表单/审批后继续                            | 暂停/恢复；sourceHandle = user_actions[].id 或 `__timeout` |
| `trigger-schedule`    | 定时触发器           | Cron/间隔触发，workflow 专属入口(ROOT)                       | 触发器(TRIGGER)                                            |
| `trigger-webhook`     | Webhook 触发器       | 收到 HTTP 回调触发，workflow 专属                            | 触发器                                                     |
| `trigger-plugin`      | 集成触发器           | 由插件事件触发，workflow 专属                                | 触发器                                                     |
| `datasource`          | 数据源               | RAG Pipeline 专用数据源入口                                  | 仅 RAG Pipeline(knowledge-index 同理)                      |
| `knowledge-index`     | 知识库               | RAG Pipeline 专用索引节点                                    | 仅 RAG Pipeline                                            |

引擎内部分类(`NodeExecutionType`):EXECUTABLE 普通执行 / RESPONSE 流式响应(answer、end)/ BRANCH 分支(if-else、question-classifier)/ CONTAINER 容器子图(iteration、loop)/ ROOT 入口(start、trigger*)。错误策略枚举:`fail-branch` / `default-value`。

---

## 2. DSL(.yml)结构要点

来源:`api/services/app_dsl_service.py`(`serialize_export_data`、`_append_workflow_export_data`)、`api/models/workflow.py`(`Workflow.to_dict`)、真实 fixture `api/services/rag_pipeline/transform/*.yml`。

顶层键:
```yaml
kind: app            # 固定 "app"(导入时缺失会被强制补上;RAG pipeline 为 rag_pipeline)
version: "0.7.0"     # DSL 版本,带引号字符串
app:                 # name / mode / description / icon / icon_background / icon_type
  mode: workflow     # 或 advanced-chat(chatflow)/ agent / chat / completion
dependencies:        # 插件依赖列表(marketplace plugin_unique_identifier),导出时自动分析生成
workflow:
  graph:
    nodes: [...]
    edges: [...]
  features: {...}         # 应用级特性(开场白、suggested questions 等)
  environment_variables: [...]   # env 变量(含 secret 类型,默认导出时 value 置空)
  conversation_variables: [...]  # 会话变量
  rag_pipeline_variables: []     # 仅 RAG pipeline
```
1.18+ 另可选 `.ifpkg` 应用包(DSL + 图标 + agent 资源)；API key、知识库数据不进 DSL。

**graph.nodes[] 每个 node(React Flow custom 包装):**
- `id`:字符串节点 id(时间戳数字)
- `type: custom`(包装器固定值；真正的节点类型在 `data.type`)
- `position: {x, y}`、`positionAbsolute`、`width/height`、`zIndex`、`selected`、`sourcePosition/targetPosition`
- `data`:`type`(节点类型字符串)、`title`、`desc`、`selected`,加上各节点专属配置。例如:
  - llm:`model`、`prompt_template`、`context: {enabled, variable_selector}`、`memory`
  - if-else:`cases: [{case_id, conditions: [{varType, variable_selector, comparison_operator, value}], logical_operator}]`
  - iteration:`iterator_input_type`、`output_type`、`is_parallel`、`parallel_nums`、`error_handle_mode`
  - loop:`loop_count`、`break_conditions`、`loop_variables`
  - tool:`provider_id/provider_name/provider_type/tool_name/tool_label/tool_parameters`
  - 迭代/循环内的子节点在同一顶层 nodes 数组里，通过 React Flow `parentId` 挂到容器节点下

**graph.edges[]:**
- `id`、`source`、`target`、`sourceHandle`、`targetHandle: target`、`type: custom`、`zIndex`
- `data: {sourceType, targetType, isInIteration, isInLoop}`(布尔标记，源码实测可见)
- `sourceHandle` 取值:普通节点为 `source`;if-else 为 case_id 或 `"false"`;question-classifier 为 class_id;human-input 为 action id 或 `__timeout`

---

## 3. 变量引用语法

来源:`graphon/src/graphon/variables/template_resolution.py`(正则原文)、`api/core/workflow/variable_prefixes.py`、`api/core/workflow/system_variables.py`。

- **模板插值**:`{{#node_id.var_name#}}`,引擎正则为
  `\{\{#([a-zA-Z0-9_-]{1,50}(?:\.[a-zA-Z_][a-zA-Z0-9_]{0,29}){1,10})#\}\}`
  即节点 id 段最长 50 位(字母数字下划线连字符)，后接 1~10 段点分变量段；用于 LLM prompt、模板转换、直接回复等文本位。
- **结构化引用(variable selector)**:节点配置里存字符串数组,如 `["node_id", "text"]`、`["sys", "query"]`、`["env", "api_key"]`。
- **变量空间前缀**(variable_prefixes.py):`sys` 系统变量 / `env` 环境变量 / `conversation` 会话变量 / `rag`(RAG pipeline)。这四个是“无节点”的虚拟前缀(ROOT_VARIABLE_NODE_IDS)。
- **系统变量**(sys.*):`query`、`files`、`conversation_id`、`user_id`、`dialogue_count`、`app_id`、`workflow_id`、`workflow_run_id`、`timestamp`,RAG/数据源相关:`document_id`、`original_document_id`、`batch`、`dataset_id`、`datasource_type`、`datasource_info`、`invoke_from`。
- **环境变量**:应用级(常存 API key 等敏感信息)，运行期**只读**;secret 类型导出 DSL 时默认置空。
- **会话变量**:跨多轮对话持久,Chatflow 专属;只能由 `assigner`(变量赋值)节点写入，`variable-aggregator`/LLM 等读取。

---

## 4. Chatflow vs Workflow 节点差异

来源:docs `workflow-chatflow.md`、`answer.md`、`output.md`;代码 `api/models/model.py` `AppMode`。

|            | Workflow(`mode: workflow`)                                   | Chatflow(`mode: advanced-chat`)                 |
| ---------- | ------------------------------------------------------------ | ----------------------------------------------- |
| 入口       | User Input(start)**或 Trigger**(schedule/webhook/plugin)     | 仅 User Input(start)                            |
| 出口       | **Output(end)** 节点，现可选；无输出节点的分支不返回数据     | **Answer(answer)** 节点，必需且可达             |
| 触发器节点 | 有(trigger-* 三种)                                           | 无(总是由用户消息触发)                          |
| 会话变量   | 不可用                                                       | 可用(配合 assigner 节点)                        |
| 流式       | Output 在 blocking 模式一次返回 `outputs`,streaming 模式随 `workflow_finished` 事件 | Answer 按变量出现顺序流式，可多处多 Answer 节点 |
| 典型用途   | 批处理、数据管道、定时任务                                   | 对话助手、引导式问答                            |

其他节点(llm、if-else、iteration、loop、code、http-request、tool、agent 等)两种模式通用。Loop 在 Chatflow 中不可用(仅 Workflow)——注意:这一点 docs 未明说，但触发器语义上 loop 属 workflow;以官方 nodes 文档页归属为准(docs 将 Loop/Iteration 列在 workflow 节点下)。

---

## 5. Iteration / Loop 执行语义

来源:docs iteration.md、loop.md;代码 `graphon/src/graphon/nodes/iteration/entities.py`、`nodes/loop/entities.py`。

**Iteration(迭代)** — 数组批量处理:
- 输入必须是数组;对每个元素执行内部子流程一次,输出为结果数组。
- 内置上下文变量:`item`(当前元素,文档写作 items)、`index`(0 起始下标)。
- 两种模式:`is_parallel` 串行(默认，顺序保证、支持 Answer 流式、省内存)或并行(`parallel_nums` 默认 10,最大 10)。
- 错误处理 `error_handle_mode` 三选一:`terminated`(立即终止报错)/ `continue-on-error`(失败位填 null:`[r1, null, r3]`)/ `remove-abnormal-output`(剔除失败项)。
- 输出数组常需 Code/Template 节点二次加工(如 join)。

**Loop(循环)** — 有状态重复:
- **循环变量** 定义在 loop 节点上,跨轮次持久保留，循环结束后仍可读；用 assigner 节点在体内更新(如计数器)。
- 三个终止机制(任一满足即停):① `break_conditions` 退出条件表达式(每轮评估)；② `loop_count` 最大循环次数(必填,ge=1,防死循环安全上限);③ 体内执行到 `loop-end`(退出循环)节点立即终止。全都不配则跑到最大次数。
- 每轮结果可反馈给下一轮(渐进精炼场景);与 Iteration 的区别:Loop 串行且有状态,Iteration 面向数组元素、元素间独立、可并行。
- 嵌套:Loop/Iteration 的子图里可以再放 Loop/Iteration(经子工作流嵌套,官方讨论 #30457)。

---

## 来源链接

源码(权威):
- [graphon enums.py — BuiltinNodeTypes / NodeExecutionType](https://github.com/langgenius/graphon/blob/main/src/graphon/enums.py)
- [graphon template_resolution.py — VARIABLE_PATTERN](https://github.com/langgenius/graphon/blob/main/src/graphon/variables/template_resolution.py)
- [graphon iteration entities](https://github.com/langgenius/graphon/blob/main/src/graphon/nodes/iteration/entities.py) / [loop entities](https://github.com/langgenius/graphon/blob/main/src/graphon/nodes/loop/entities.py)
- [dify api/core/workflow/variable_prefixes.py](https://github.com/langgenius/dify/blob/main/api/core/workflow/variable_prefixes.py) / [system_variables.py](https://github.com/langgenius/dify/blob/main/api/core/workflow/system_variables.py)
- [dify api/constants/dsl_version.py](https://github.com/langgenius/dify/blob/main/api/constants/dsl_version.py) / [api/constants/trigger.py](https://github.com/langgenius/dify/blob/main/api/constants/trigger.py)
- [dify api/models/model.py — AppMode](https://github.com/langgenius/dify/blob/main/api/models/model.py) / [api/models/workflow.py — to_dict](https://github.com/langgenius/dify/blob/main/api/models/workflow.py)
- [dify api/services/app_dsl_service.py — DSL 导出/导入](https://github.com/langgenius/dify/blob/main/api/services/app_dsl_service.py)
- [1.9.0 legacy NodeType 枚举](https://github.com/langgenius/dify/blob/1.9.0/api/core/workflow/enums.py)
- [zh-Hans 节点中文名 workflow.json](https://github.com/langgenius/dify/blob/main/web/i18n/locales/zh-Hans/workflow.json)

官方文档:
- [节点总览](https://docs.dify.ai/en/cloud/use-dify/nodes/) / [Iteration](https://docs.dify.ai/en/cloud/use-dify/nodes/iteration) / [Loop](https://docs.dify.ai/en/cloud/use-dify/nodes/loop) / [Answer](https://docs.dify.ai/en/cloud/use-dify/nodes/answer) / [Output(原 End)](https://docs.dify.ai/en/cloud/use-dify/nodes/output) / [Human Input](https://docs.dify.ai/en/cloud/use-dify/nodes/human-input) / [Trigger 总览](https://docs.dify.ai/en/cloud/use-dify/nodes/trigger/overview)
- [Workflow vs Chatflow](https://docs.dify.ai/en/self-host/use-dify/build/workflow-chatflow.md) / [Key Concepts(DSL 说明)](https://docs.dify.ai/en/learn/key-concepts) / [应用导出导入](https://docs.dify.ai/en/guides/management/app-management)
- [变量赋值器(会话变量)](https://docs.dify.ai/zh/cloud/use-dify/nodes/variable-assigner)

第三方实用参考(非官方，已与源码交叉验证):
- [dify-workflow-dsl-skill SKILL.md](https://github.com/yzmw123/dify-workflow-dsl-skill/blob/main/SKILL.md)
- [嵌套循环讨论 #30457](https://github.com/langgenius/dify/discussions/30457)

Sources:
- [Loop - Dify Docs](https://docs.dify.ai/en/cloud/use-dify/nodes/loop)
- [Iteration - Dify Docs](https://docs.dify.ai/en/cloud/use-dify/nodes/iteration)
- [Workflow vs Chatflow - Dify Docs](https://docs.dify.ai/en/self-host/use-dify/build/workflow-chatflow.md)
- [Key Concepts - Dify Docs](https://docs.dify.ai/en/learn/key-concepts)
- [graphon enums.py](https://github.com/langgenius/graphon/blob/main/src/graphon/enums.py)
- [dify app_dsl_service.py](https://github.com/langgenius/dify/blob/main/api/services/app_dsl_service.py)
- [dify-workflow-dsl-skill](https://github.com/yzmw123/dify-workflow-dsl-skill/blob/main/SKILL.md)
- [变量赋值器 - Dify Docs](https://docs.dify.ai/zh/cloud/use-dify/nodes/variable-assigner)
- [Dify Conversation Variables 博客](https://dify.ai/blog/dify-conversation-variables-building-a-simplified-openai-memory)

---

## 勘误（2026-11-26 源码复核）

以下结论由子代理对照 Dify 与 graphon 源码复核得出，原文保留未改。完整时间线与来源见 docs/grill-record.md 的 Round 6 结论。

1. **「End 节点已更名为 Output」不成立。** End 节点的 type 字符串在 graphon main（94109ce8，2026-11-25）和 Dify 1.9.0 里都是 `end`，不存在 `output` 类型。「输出」是否只是界面上的显示名未核实。
2. **「节点 type 字符串在 1.9.0 → main/graphon 之间完全一致」需要限定范围。** graphon 的 BuiltinNodeTypes 有 25 个成员，自首次提交未变；与 Dify 1.9.0 的差别是 1.9.0 有 `knowledge-index`、没有 `human-input`，没有共同的字符串被改过。trigger-* 不在 graphon 里，在 `api/constants/trigger.py`。
3. **上表不是 Dify main 的最新全集。** 0.7.0 之后没有升版本号就加了 `site`、LLM 的 `model_selector`、human-input v2、`knowledge-retrieval-v2`、`agent-v2`，以及默认的 `.ifpkg` 导出（2026-09-20，zip 包，内含 manifest、DSL 和资源；显式 YAML 导出仍存在）。这些的确切形态待进一步核查。
4. **版本号。** 0.7.0 由 1.16.0（2026-07-17）引入，1.20.0 与 main 仍是它；0.6.0 由 1.13.0（2026-02-11）引入。完整时间线见 grill-record。
5. **版本号和内容不可靠地对应。** 节点 schema 在小版本里变过，也有不升版本号就改的；按版本号把关没有意义，要按节点 `data.type` 对照引擎支持的类型来校验。Dify 自己的导入器不拒绝任何旧版本 DSL。
6. **LLM 节点的 memory。** 配置是 `role_prefix`、`window`（enabled、size）、`query_prompt_template`。历史来自该会话已存的 `messages`（query 作用户轮、answer 作助手轮），不经会话变量或环境变量；没有 `sys.conversation_id` 的 workflow 应用没有记忆；上限 500 条消息、2000 tokens。
7. **human-input 没有审批人模型。** 谁拿到表单 token 谁就能提交；`delivery_methods` 的接收人只是投递目标。