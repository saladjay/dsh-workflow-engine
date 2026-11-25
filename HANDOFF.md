# HANDOFF — dsh-workflow-engine 设计阶段交接

> 写于 2026-11-26。本仓库目前只有设计文档，没有代码。
> 交接原因：dsh 源码不在写这份文档的机器上。计划是本仓库推到 GitHub，在另一台有 dsh 源码的机器上拉取后继续。
> 远端：`git@github.com:saladjay/dsh-workflow-engine.git`，main 分支，PUBLIC（用户确认公开是有意的）。后续改动继续推送到这里。

## 1. 这是什么

- **项目**：企业版 dsh 的 workflow 执行底层，重写为 Rust 独立服务（数据面）；dsh（Node）只做控制面。要解决的痛点：Node 单线程跑并行 workflow 效率差；隔离要从会话级下沉到 call-id 级。
- **目标能力**：Dify DSL 导入执行、HTTP 接口快速封装成节点（Capability）、国内云厂商兼容、结构化输出、全量落库与断点续跑、token 统计、工作流能产出文件（图片、Word、PPT、Excel）。
- **产品定位**（用户原话，影响很多设计）：工作流目前是 dsh 的 agent 看得到的一个工具；一个 dsh 会话可以调用多个工具，其中可以包括多个工作流。

## 2. 现在到哪了

- 设计用 grill 的方式做（逐轮追问），已进行 9 轮，决策编号 D1–D63，全部在 `docs/grill-record.md`。
- Round 8 已答完。Round 9 的三题（第 5 节）已问出，还没答；第四题等事实。**没有开始写代码。**
- `docs/` 下的四份交付文档（api-spec、db-schema、dify-node-mapping、milestones）是 Round 4 时的初稿，**已知过时**，计划等设计树清空后统一修订。以 `docs/grill-record.md` 和 ADR 为准。过时点清单见第 6 节。

## 3. 文件地图

| 文件 | 内容 | 可信度 |
|---|---|---|
| `docs/grill-record.md` | 每一轮的原问题、推荐答案、用户原话回答、决策表 D1–D63、事实核查结果 | 最高。设计问题先看这里 |
| `docs/adr/0001–0014` | 14 条架构决策，早期的带「修订」小节 | 高。被修订的旧表述在修订小节里说明 |
| `GLOSSARY.md` | 领域术语 | 高。术语定了就更新 |
| `docs/open-issues.md` | 待补充的事实（OI-001） | 高 |
| `dify node.md` | Round 2 时的 Dify 调研报告，文末有「勘误」 | 中。勘误之前的内容有错，以勘误和 grill-record 的「核查结果」为准 |
| `dify history.md` | 子代理对 Dify DSL 版本史的原始核查报告（英文），含各次升版的提交号和 PR 号 | 高。比 grill-record 里的摘要更细 |
| `docs/api-spec.md`、`docs/db-schema.md`、`docs/dify-node-mapping.md`、`docs/milestones.md` | 四份交付初稿 | 低。只改过几处直接矛盾，其余过时 |

## 4. 怎么干活（约定）

- **方式**：grill-with-docs 技能（＝ grilling + domain-modeling，装在用户的 `~/.agents/skills` 下）。一轮把当前前沿的问题问完，每题给推荐答案，问题措辞让「同意推荐」即可通过；用户答完一轮再算下一轮。**事实自己查（派子代理），决策留给用户。**
- **记录约定**：`docs/grill-record.md` 里原问题、全部选项、推荐答案、用户原话回答逐字保留，不缩写。用户答完一轮，才把这一轮写进去。
- **编号**：决策 D 连续编号。每轮问题用一个字母前缀：Round 1 是 Q，2 是 R，3 是 S，4 是 T，5 是 U，6 是 V，7 是 W，8 是 X，9 是 Y。
- **文档分工**：GLOSSARY.md 只放术语，不放实现细节。ADR 只在「难逆转、缺上下文会费解、真有权衡」三条都满足时才写，能改就改旧的并加「修订」小节。`docs/open-issues.md` 放待补充的事实。
- **默认处理**：有常规做法的细节不单独出题，列成「默认处理（不同意再说）」清单，用户不提异议即生效。
- **用户的习惯**：回答很短，多数是「同意推荐」；会漏答个别题，下一轮开头再提一次即可；同一件事不要反复追问；用中文。
- **教训**：Round 4 收尾时写过「前沿为空」，复审时发现不成立。别急着宣布设计树走完，把决策逐条对着 schema 和 API 核一遍再说。

## 5. 没做完的

### Round 9 待答（前沿）

原题在 `docs/grill-record.md` 的 Round 9。

1. **Y1** 「Office 文档生成和编辑」这个功能节点以什么形态出现：Capability + Http-Tool Node，还是专用节点类型。
2. **Y2** 文件怎么在引擎和外部服务之间传：内联 base64、预签名地址，还是服务直连文件存储。
3. **Y3** 文档节点本期支持哪些操作：模板填充、从零生成、编辑既有文件、格式转换。
4. **Y4** 实现语言与调用方式（C# 独立服务、Python worker、Rust 内置）。依赖 F7、F8，事实到了再问。

### 待事实

- **dsh 现有工作流定义模型**（F3）。到了有源码的机器上要查：画布存成什么格式、有哪些节点类型、怎么触发、会话（session）模型、ask-user-question 类插件的实现。查完接着做「dsh 接入与存量迁移」这一轮：转换器怎么写、迁移策略、dsh 侧的里程碑。
- **Dify 0.7.0 之后未升版本号的新增**（F6）：`knowledge-retrieval-v2`、`agent-v2`、human-input v2、`site`、LLM `model_selector` 的确切形态；Dify 的工作流总执行时长与步数上限；除 LLM 外哪些节点带 memory；human-input 的 timeout 语义；`.ifpkg` 的包结构。
- **Rust 的 Office 编辑库**（F7）：xlsx / pptx / docx / pdf 的读、生成、改现有文件各有哪些 crate。
- **C# 的 Office 库与 Rust 调 C# 的方式**（F8）：各库的能力和许可证；独立服务、进程池、hostfxr 进程内、NativeAOT 动态库各自的调用开销、隔离、部署、复杂度。
- **OI-001** 企业网关能否替换请求 body 里的密钥占位符。要问网关侧。

F6、F7、F8 在推送时各有一个子代理在查。**结果如果没写进 grill-record，就需要重查**；上面列的条目就是重查的提纲。

### 待动作

- **统一修订四份交付文档**，并重排里程碑：加 dsh 接入线、Provider 配置、发布生命周期、准入控制、文件存储与产出、文档节点。
- **dsh 侧的 grill**（见上面的 F3）。

## 6. 四份交付文档里已知的过时点

**api-spec.md**

- `POST /invocations` 的 `timeout_ms`：只累计运行中的时间，等用户输入和暂停期间不计；没有工作流级的提醒时限。状态查询要支持长轮询。
- `/pause`、`/resume` 的语义按 D49；human-input 有提醒时限和对应的提醒事件。
- 管理 API 缺：用 Workflow Definition 新建版本、更新草稿、发布（D53、D59）；Provider 配置的增删改查（D35）；文件的上传、下载、签名链接（D61）。

**db-schema.md**

- `workflow_versions.dsl_yaml NOT NULL`（D33）；缺 draft / published 状态和发布时冻结的绑定（D53、D54）。
- `node_runs`、`waiting_inputs`、`node_snapshots` 的标识要加 Node Run（D38）；`waiting_inputs` 加 `assignees`（D45）。
- `events`：流式增量不落库（D37）。
- `invocations`：缺轮次（D56）、共享配置的快照（D54）。
- `conversation_variables` 要带 workflow 维度（D50）。
- 缺 Provider 配置表（D35）、文件表（D61，细节待 Y2）。
- `secrets` 表只在 local 部署档位存在（D43）。
- 第 7 节「多实例」的说法不成立（D47）。
- 没有留存策略（D63）。

**dify-node-mapping.md**

- 节点全集待 F6：0.7.0 之后又有新增。
- LLM 行没有体现 memory 由引擎提供（D56）、Provider 配置与导入期模型映射（D35）。
- 知识检索导入后默认是阻断项；环境变量、`features` 的处理没写。
- 自有节点里没有文档节点（D62）。

**milestones.md**

- 缺 dsh 接入线；Node Binding 排在 M6 太靠后，M4 就用到了。
- 缺 Provider 配置、发布生命周期、准入控制、文件存储与产出、文档节点。
- 缺 Dify 兼容的一致性测试夹具。

## 7. 影响最大的决策（速查）

| 决策 | 内容 |
|---|---|
| D1–D3 | Rust 独立服务做数据面，dsh 只做控制面；按 Invocation（call-id）隔离，带幂等键和租户 |
| D33（ADR-0010） | 原生的工作流定义以 Dify DSL 0.7.0 的结构为底，加扩展命名空间；内部 IR 只是派生物 |
| D34（ADR-0013） | 执行用的配置只存引擎库，控制面经引擎的管理 API 读写 |
| D36（ADR-0011） | 重启后在途节点至少一次重跑，幂等键随节点执行下发 |
| D37（ADR-0012） | 事件只在节点边界落库，流式增量不落库 |
| D38 | 每次节点执行有 `node_run_id`，容器本身也是一次节点执行 |
| D42、D60 | Dify DSL 接受 0.1.5–0.7.0 并做导入期归一化；更新的文件带警告导入；接受 `.ifpkg`。下限不是硬需求 |
| D43（ADR-0008） | 密钥只有 `{{secret.name}}` 一种写法；出口按部署档位选网关或本地；MCP 只接远程传输 |
| D47 | 本期只支持单实例 |
| D50（ADR-0007） | Conversation 绑定单一工作流，与 dsh 的 Session 是两个概念 |
| D53（ADR-0014） | 导入严重度三级；版本分草稿与已发布，只有已发布的能被调用 |
| D54（ADR-0014） | 绑定在发布时冻结；共享运行配置在调用开始时解析并留快照 |
| D61 | 引擎有可插拔的文件存储，节点可以产出文件；File 一律引用传递 |
| D62 | Office 文档的生成和编辑收拢成一个功能节点，实现可考虑 C#（细节在 Round 9） |

## 8. 环境

- **参考库**：pi_agent_rust、rig、swiftide、graphbit 是写这份文档的机器上 `E:/github` 下的 submodule，另一台机器上未必有。graphbit 是 source-available 许可，只作架构参考，法务确认前不引用、不复制代码。
- **Dify 的事实来源**：Dify 与 langgenius/graphon 的源码，核查结果在 grill-record 的 Round 6 结论里。调研报告 `dify node.md` 里被推翻的说法列在它的「勘误」里。
- **日期**：设计做于 2026-11-25 至 2026-11-26。Dify 当时稳定版 1.20.0，DSL 版本 0.7.0。
