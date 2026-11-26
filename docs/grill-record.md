# Grill 过程完整记录 — 企业版 dsh Workflow 执行底层重设计

> 项目：dsh-workflow-engine（企业版工作流执行底层）
> 记录约定：完整保留每轮原问题、全部选项、推荐答案与用户的原始回答，不做缩写概括。
> 状态标记：`[已定]` = 用户已拍板；`[待决]` = 已展开待回答；`[事实待查]` = 等子代理/调研结果。
> ⚠️ 2026-11-26 复审：Round 4 结束语中的「前沿为空」不成立，设计树在 Round 5 重开。D8 / D10 / D11 / D13 / D14 / D17 / D23 / D24 / D28 / D31 已被修订，对照见 Round 5 结论。

---

## Round 0 — 需求原话（用户原始陈述，2026-11-25）

> 我有个基于dsh的workflow，你可以先不看源码，dsh被我们改成企业版，但是node v8引擎是个单线程的，跑workflow会有很多问题，例如并行的workflow，比如多用户多工作流并行的效率问题。我希望引入一个更好的workflow执行底层，需要你来协助我完成设计，抬高设计上下限。首先workflow执行是会话级别的隔离，我希望更近一层到调用call-id的隔离，workflow需要兼容映射所有dify的节点，workflow有能执行http接口的能力，大模型输出结构化的能力，兼容国内云厂商，数据存储可以存数据库，统计token的能力，快速封装http接口形成新功能节点的能力（或者同一种节点，下拉框选择功能，通过工厂设计模式派生，不写死到代码里，快速配置），目前我能想到就那么多

**背景补充（本轮对话前已发生）**：已将 pi 的 Rust 重写（pi_agent_rust）及三个 Rust/C++ agent 库（swiftide、rig、graphbit）以 git submodule 形式下载到 E:/github 下，作为候选依赖参考。

---

## Round 1 — 第一轮前沿问题（2026-11-25）

### ❓ Q1 - 执行底层的形态（语言 + 部署方式）

新的 workflow 执行底层以什么形态存在？
(a) **Rust 独立服务**（dsh 通过 HTTP/gRPC 调用）　(b) Rust 通过 NAPI 内嵌进 Node 进程　(c) Go 独立服务　(d) 纯 Node 重构（worker_threads / cluster）

➡️ 推荐答案：(a) Rust 独立服务：Tokio 多线程直接解决并行 invocation 问题；进程级崩溃隔离；可独立水平扩展。不推荐 (b)：NAPI 和 V8 生命周期/事件循环绑死，重活还是会互相干扰；不推荐 (d)：治标不治本。已下载的 rig/swiftide/graphbit 都能直接当依赖用。

**用户回答：A** ✅ [已定] → **决策 D1：执行底层 = Rust 独立服务**（dsh 经网络协议调用）。

---

### ❓ Q2 - dsh 与新引擎的职责边界

改造后 dsh（Node 企业版）负责什么？
(a) dsh 只做**控制面**（工作流定义、画布、触发入口、查询结果、权限），执行全部委托新引擎　(b) 简单流程留在 dsh 本地跑，复杂/并行的走新引擎　(c) 新引擎彻底取代 dsh

➡️ 推荐答案：(a) 控制面/数据面分离是标准做法，dsh 现有的画布和触发逻辑全部保留，只把"执行器"替换掉。(b) 会导致两条执行语义长期漂移，维护噩梦。

**用户回答：A** ✅ [已定] → **决策 D2：dsh = 控制面，新引擎 = 数据面**。

---

### ❓ Q3 - call-id 隔离的精确语义

你说"会话级隔离要拉近到 call-id 级"。提议定义：**Invocation（调用实例）**= 某工作流的一次具体执行，`invocation_id`（即 call-id）为隔离边界——变量上下文、内存、超时、取消、日志、token 统计全部按 invocation 隔离，同一工作流任意多 invocation 并行互不干扰。需要确认两点：① call-id 由**引擎生成**还是由**上游业务传入**（做幂等/追踪）？② 多用户的"用户"是否要升级为**租户（tenant）**维度挂在 invocation 上？

➡️ 推荐答案：引擎生成 `invocation_id` + 支持外部传 `idempotency_key`（幂等键，重复提交返回同一实例）；引入 `tenant_id / user_id` 作为 invocation 的必填元数据（隔离与统计都按它切分）。

**用户回答：同意推荐，保留隔离的情况下又能联动** ✅（部分）→ **决策 D3：Invocation 隔离模型 + 引擎生成 invocation_id + 幂等键 + tenant_id/user_id 元数据。**
⚠️ **新增待决分支："联动"的具体语义未定义**（隔离的前提下 invocations 之间/与外部如何交换数据）→ 见 Round 2 R1。

---

### ❓ Q4 - Dify 兼容的目标层次

"兼容映射所有 Dify 节点"做到哪一层？
(a) 仅**语义兼容**（节点类型和执行语义对齐，DSL 格式无关）　(b) **DSL 级兼容**：能原样导入 Dify 导出的 .yml 并正确执行（含 `{{#node.var#}}` 变量语法的映射）　(c) 只借鉴其节点分类学，不承诺兼容

➡️ 推荐答案：(b) 企业客户的存量 Dify 工作流是真实迁移需求，DSL 导入是硬卖点；但不承诺反向导出（dsh 特色功能无法表达成 Dify DSL）。

**用户回答：B** ✅ [已定] → **决策 D4：Dify DSL 级导入兼容（含变量语法映射），不做反向导出。**
⚠️ 衍生待决：DSL 基线版本与不支持节点的导入策略 → 见 Round 2 R4。Dify 节点全集清单 [事实待查]（子代理调研中）。

---

### ❓ Q5 - 国内云厂商兼容策略

兼容方式选哪种？
(a) 统一走 **OpenAI 兼容协议**为主干（Dashscope/DeepSeek/Moonshot/智谱/豆包都提供 OpenAI 兼容端点），个别特殊协议（百度千帆旧版、讯飞星火）做适配器　(b) 每家原生 SDK 逐个对接　(c) 只做 OpenAI 兼容，特殊厂商不接

➡️ 推荐答案：(a) 一个 `Provider` trait + 配置驱动（base_url/api_key/model/协议模式），新厂商=加一条配置而非写代码。理由：国内主流厂商 90% 能力 OpenAI 兼容端点已覆盖。

**用户回答：A** ✅ [已定] → **决策 D5：OpenAI 兼容协议主干 + 特殊厂商适配器，配置驱动。**

---

### ❓ Q6 - 结构化输出的实现策略

(a) 只靠 provider 原生能力（JSON mode / json_schema）　(b) 只靠提示词 + 自己解析　(c) **分层**：优先原生 JSON mode → 不支持时降级为提示词约束 + JSON Schema 校验 → 校验失败自动重试（把校验错误注入提示词，最多 N 次）

➡️ 推荐答案：(c) 结构化输出不能赌厂商能力，分层降级是唯一在所有国内厂商上都稳定的做法。Schema 定义放在 LLM 节点配置里（JSON Schema 标准）。

**用户回答：C** ✅ [已定] → **决策 D6：分层结构化输出（原生 JSON mode → 提示词+Schema 校验 → 校验错误注入重试 N 次）。**

---

### ❓ Q7 - 数据库存储的范围与选型

存什么 + 用什么库？
(a) 只存运行记录（invocation 状态、节点输出、token 统计）　(b) (a) + 工作流 DSL 定义及版本　(c) (b) + 每节点变量快照（支持断点恢复、审计）
选型：MySQL / PostgreSQL / 都要

➡️ 推荐答案：(c) + 抽象仓储接口，MySQL 驱动优先（国内企业环境 MySQL 最普遍，PG 作为第二驱动）。每节点完成即落库，是暂停/恢复和审计的地基。

**用户回答：C** ✅ [已定] → **决策 D7：全量存储 = 运行记录 + DSL 定义及版本 + 每节点变量快照；仓储抽象接口，MySQL 优先。**

---

### ❓ Q8 - 长任务能力

工作流需要支持**暂停/恢复**（人工审批节点）、**取消**、以及**进程重启后续跑**吗？
(a) 本期就要三者　(b) 只要取消，暂停/续跑后置　(c) 都不要，工作流都是秒级的

➡️ 推荐答案：(a) 企业版几乎必然遇到"审批后继续执行"场景；而"每节点落库"（Q7c）做完后，续跑只是加一个恢复入口，边际成本很低。

**用户回答：A，我的工作流里有类似ask-user-question的插件功能的节点，用户可以输入内容或者审批执行** ✅ [已定 D8，展开新需求] → **决策 D8：暂停/恢复/取消/进程重启续跑全部为本期范围；且必须支持 ask-user 类"等待用户输入/审批"节点。**
⚠️ 衍生待决：human-input 节点的暂停/唤醒机制细节 → 见 Round 2 R2。

---

### ❓ Q9 - HTTP 快速封装成节点的机制

提议定义为**端点注册表（Capability Registry）**：管理员把一个 HTTP API（method/URL/headers/鉴权/输入输出 schema/超时/重试）注册为一个**能力（Capability）**，存数据库；节点类型只有一种 `http-tool`，配置里下拉选 capability，执行器用工厂按 capability 配置派生实际执行器。新增功能=注册一条记录+热加载，不改代码、不发版。
本期注册表的管理方式：(a) 数据库手工配置 + 管理接口　(b) 完整管理 UI 界面　(c) 配置文件（yaml）热加载

➡️ 推荐答案：(a) 本期数据库 + 管理 API（供 dsh 后台页面后续对接），UI 后置。鉴权支持 Bearer/API Key/Header 自定义即可覆盖大部分场景。

**用户回答：A** ✅ [已定] → **决策 D9：Capability Registry 存数据库，本期提供管理 API；http-tool 单一节点类型 + 工厂按 capability 派生；新增能力零代码改动。**

---

### ❓ Q10 - 性能目标（决定要不要分布式）

多用户多工作流并行，预期的量级是？
(a) 单机 <1k 并发 invocation，QPS < 100　(b) 1k~1w 并发，需要多实例 + 队列　(c) 更大规模，多区域部署

➡️ 推荐答案：(a) 为设计目标，但架构上预留 (b) 的扩展点：执行器做成"worker 从队列拉任务"的模式——单实例时队列在内存，多实例时换 Redis/消息队列，代码结构不变，只需换队列实现。

**用户回答：A（目前），同意你的说法** ✅ [已定] → **决策 D10：设计目标单机 <1k 并发 invocation；执行器采用 worker-拉队列模式，队列抽象化（内存 → Redis/MQ 可换），预留多实例扩展。**

---

## Round 1 结束后的已定决策汇总

| # | 决策 | 来源 |
|---|---|---|
| D1 | 执行底层 = Rust 独立服务 | Q1 |
| D2 | dsh 控制面 / 新引擎数据面 | Q2 |
| D3 | Invocation 隔离 + 幂等键 + tenant/user 元数据（"联动"语义待决） | Q3 |
| D4 | Dify DSL 级导入兼容，不做反向导出 | Q4 |
| D5 | OpenAI 兼容主干 + 特殊厂商适配器 | Q5 |
| D6 | 分层结构化输出（原生→提示词+校验→注入重试） | Q6 |
| D7 | 全量存储 + 每节点落库；仓储抽象，MySQL 优先 | Q7 |
| D8 | 暂停/恢复/取消/续跑 + ask-user 类节点本期必做 | Q8 |
| D9 | Capability Registry 数据库 + 管理 API，零代码派生节点 | Q9 |
| D10 | 单机 <1k 并发目标，队列抽象预留多实例 | Q10 |

## 未决分支（Round 2 前沿）

1. **R1 "联动"语义**：隔离前提下，invocation 与外部（会话、其他 invocation）如何交换数据
2. **R2 human-input 节点机制**：暂停/唤醒的 API 形态、超时策略、审批与输入的节点建模
3. **R3 引擎入口 API 形态**：同步/异步/事件流（SSE vs WebSocket）
4. **R4 Dify DSL 兼容边界**：基线版本、不支持节点导入策略（依赖子代理节点清单报告）
5. **R5 变量系统**：引用路径语法、类型校验、大对象（文件/图片）传递
6. **R6 Code 节点执行方案**：Dify 的 Code 节点（Python3/JS）在 Rust 引擎里如何沙箱执行
7. **R7 安全**：引擎 API 鉴权、HTTP 节点的 SSRF 防护
8. **R8 token 统计维度与成本换算**
9. **R9 节点错误处理模型**：重试/继续/失败传播策略

---

## Round 2 — 第二轮前沿问题（2026-11-25）

### ❓ R1 - "联动"语义（D3 的待决分支）+ 子工作流

隔离前提下，"联动"指什么？
(a) **继承注入**：invocation 启动时从会话/上游拷贝一份上下文快照（只读继承，写不回）
(b) **显式共享存储**：invocation 之间不能直接读写对方变量，但通过租户级共享层（共享变量/数据库）按 key 显式交换
(c) **直接互操作**：同租户 invocation 可以互相读写变量、等待对方

➡️ 推荐答案：(a)+(b)。默认继承快照；跨 invocation 交换只走显式共享层。(c) 会重新引入共享可变状态，把隔离打散——若场景是"并行分支跑子流程"，应建模为**子工作流**（父 invocation 派生子 invocation、结果回传父级），而不是平级互读写。因此追加确认：**子工作流是否本期需求？**（Dify 无原生子工作流节点，企业场景常见，建议建模为节点类型 `sub-workflow`，本期做。）

**用户回答：** 同意推荐

---

### ❓ R2 - human-input（ask-user 类）节点机制

执行到该类节点时 invocation 挂起，等待用户输入/审批。三个子决策：
① **唤醒通道**：(a) 异步事件：启动请求立即返回 invocation_id，用户输入经控制面 API 提交，引擎被事件唤醒续跑　(b) 同步挂起：启动请求长连接挂住直到用户响应　(c) 异步为基本机制 + 控制面可选封装长轮询，制造同步体感
② **节点建模**：一个节点类型 `human-input`，模式字段区分：审批（approve/reject）/ 表单输入（字段 schema 可配置）/ 审批+输入混合；输出变量 = 用户提交内容；超时可配置，超时走 timeout 分支
③ **挂起持久化**：挂起时快照落库，进程重启后唤醒依然有效

➡️ 推荐答案：①(c)　② 单节点+模式字段　③ 是（与 ADR-0003 一致）。

**用户回答：** 同意推荐

---

### ❓ R3 - 引擎入口 API 形态

dsh（控制面）调用新引擎的接口风格：
(a) 仅异步：提交返回 invocation_id，控制面轮询查状态
(b) **异步提交 + SSE 事件流**：事件模型 = 节点开始/节点完成/节点失败/LLM 输出增量/等待输入/终态（完成/失败/取消）；事件同步落库
(c) 同步 + 异步都提供

➡️ 推荐答案：(b)。SSE 单向推送够用且对企业代理/网关友好（WebSocket 需协议升级，内网代理环境常出问题）；LLM 流式输出天然是事件增量；不提供同步挂起模式——与断点续跑矛盾，控制面可用 SSE 达成同步体感。

**用户回答：** B

---

### ❓ R4 - Dify DSL 兼容边界（基线与不支持节点策略）

① 基线版本：以 Dify 当前稳定版 DSL schema 为基线（具体版本号等节点清单调研报告返回后确定），历史版本不承诺
② **不支持节点导入策略**：(a) 整单导入失败　(b) **占位导入**：生成 `unsupported` 占位节点，保留拓扑，导入报告明确列出，执行到该节点时明确报错　(c) 静默丢弃
③ **应用类型范围**：workflow（自动执行型）与 chatflow（对话型，含 answer 流式输出节点）都支持？

➡️ 推荐答案：②(b)——存量迁移常见"99% 节点可用"，整单拒绝体验差，占位保留拓扑、替换后即可跑。③ 都支持，chatflow 的 answer 节点映射为流式输出事件。

**用户回答：** 同意推荐

---

### ❓ R5 - 变量系统设计

① 引用路径：兼容 Dify 的 `{{#node_id.output.path#}}` 语法（DSL 导入时转换为内部结构化引用，内部统一用结构化引用执行）
② **类型校验**：(a) 弱类型 JSON 值，不校验　(b) 弱类型 + 节点输出可选 schema 标注 + **导入时静态校验所有引用可解析**（引用了不存在的节点/路径在导入时就报错）
③ **大对象（文件/图片/长文档）传递**：(a) 值传递：变量里放全量内容　(b) **引用传递**：上传对象存储/文件服务，变量带 uri + mime + 大小元数据

➡️ 推荐答案：②(b)　③(b)——文档提取器/图片类节点如果值传递，内存和数据库都会爆。

**用户回答：** 同意推荐

---

### ❓ R6 - Code 节点执行方案

Dify 的 Code 节点支持 Python3 / JavaScript 用户代码。Rust 引擎里的执行方案：
(a) **JS 进程内嵌池**（rquickjs/deno_core）+ **Python 外部沙箱进程池**（每执行独立进程，CPU/内存/超时资源限制，超时强杀）
(b) 全部容器沙箱（K8s 环境每执行一个容器）
(c) 本期不支持 code 节点（导入占位）

➡️ 推荐答案：(a)。JS 进程内池成本低；Python 无法嵌入 Rust，用进程池+资源限制。容器沙箱作为后续"企业安全档位"可选开启，本期以企业内网可信代码为假设。

**用户回答：** 【用户反问】"有可能提供rust调用的python沙箱吗，怎么解决python环境的问题" → 技术答复已给出（见 Round 2 结论附录），待用户确认后定案

---

### ❓ R7 - 安全基线（引擎鉴权 / SSRF / 密钥存储）

① 引擎 API 鉴权：(a) 内网 + 服务级 API Key/JWT（控制面持有）　(b) mTLS 双向证书
② HTTP 节点 SSRF 防护：Capability 注册与执行时校验目标地址——内网段/云元数据地址黑名单 + 可配置白名单
③ Capability 密钥存储：AES-GCM 加密落库，主密钥来自环境变量/KMS

➡️ 推荐答案：①(a) 起步、预留 (b)；②③ 属于企业级"必做的正确事"，直接做。此题基本是确认题。

**用户回答：** 同意推荐

---

### ❓ R8 - token 统计与成本

① **明细维度**（每节点每次 LLM 调用一条记录）：tenant_id, user_id, invocation_id, node_id, provider, model, prompt_tokens, completion_tokens, total_tokens, 耗时, 状态, 时间
② **成本换算**：(a) 只记 token 数，价格表后置　(b) 配置驱动的价格表，**成本在查询时按价格表版本换算**（明细表不存成本值——价格随时变，落库值会失真，查询换算可追溯价格版本）

➡️ 推荐答案：① 必做；②(b) 查询时换算。

**用户回答：** ①必做 ② 记录token和模型/provider就好

---

### ❓ R9 - 节点错误处理模型

(a) **完全对齐 Dify 语义**：节点级配置 重试次数/重试间隔 + 异常处理策略（none / fail branch 异常分支 / default value 默认值兜底），保证 DSL 导入后行为一致
(b) 只做全局 fail-fast + 节点重试

➡️ 推荐答案：(a)。D4 已定 DSL 级兼容，错误处理语义是执行语义的一部分，必须对齐；否则导入的工作流行为与 Dify 不一致，兼容承诺不成立。

**用户回答：** 同意推荐

---

### ❓ R10 - 会话（Conversation）与会话变量 【新事实解锁】

[事实] Dify chatflow 有 conversation_variables（跨轮持久，仅 assigner 节点可写）和 sys.conversation_id 系统变量。
① 术语定义：**Conversation（会话）**= 多轮对话中多个 Invocation 的容器；chatflow 每次用户消息触发一个 Invocation，挂在 Conversation 下。
② 职责切分：Conversation 生命周期与 conversation_id 由控制面（dsh）管理并传入引擎；引擎负责会话变量的读写（assigner 节点）与按 conversation_id 持久化。

➡️ 推荐答案：引入 Conversation 实体，conversation_variables 存引擎 DB。这也正是 Q3"保留隔离又能联动"的合法联动通道——会话变量按 conversation 隔离、invocation 读写时受控。

**用户回答：** 同意推荐

---

### ❓ R11 - 触发器节点归属 【新事实解锁】

[事实] Dify 新增三种 workflow 专属入口节点：trigger-schedule（定时）/ trigger-webhook（HTTP 回调）/ trigger-plugin（插件事件）。
问题：触发器归谁管？
(a) **归控制面**：dsh 负责定时调度、webhook 接收、权限与审计，引擎只暴露统一的 API 触发入口；DSL 导入时 trigger-* 节点映射为 dsh 的触发配置
(b) 引擎内建调度器与 webhook 接收器（引擎自治触发）

➡️ 推荐答案：(a)。触发入口唯一化，权限/审计/限流全部在控制面；引擎保持纯粹执行。

**用户回答：** 同意推荐

---

### ❓ R12 - tool / 知识检索 / agent 三类"生态型"节点的支持范围 【新事实解锁】

[事实] Dify 的 tool 节点引用插件工具（provider_type: builtin / api / workflow / mcp）；另有 knowledge-retrieval（知识检索）、agent（Agent 策略）节点。
分级提议：
- **tool + provider_type=api** → 直接映射 **Capability Registry**（与 D9 天然契合，全兼容）
- **tool + provider_type=mcp** → 引擎实现 MCP 客户端接工具（本期做；与 pi/goose 生态一致）
- **tool + builtin**（内置工具如 web search）→ 按需实现常用子集，其余占位
- **knowledge-retrieval** → 本期实现基础版，对接企业现有向量库（**需要你告知：企业现有向量库/知识库是什么？** Qdrant/Milvus/Elasticsearch/自研？）
- **agent 节点** → 本期实现基础版（LLM + 工具循环，策略配置化），复杂 agent 策略后置

➡️ 推荐答案：按上述分级执行。

**用户回答：** 同意推荐，向量库/知识库通过http接口对接，直接写成功能节点，另外可以通过http对接其他外部知识库。引擎要和控制面沟通xtoken等变量，我们预计传入一个占位符，这个请求出网关时进行替换

---

### Round 2 事实报告摘要（[事实已查]，子代理调研，2026-11-25）

影响设计的关键事实：
1. **DSL 版本基线 = "0.7.0"**（Dify 1.16+，当前稳定版 1.20.0；2.0.0-beta 已打 tag）——R4 基线确定
2. **End 节点已更名 Output（输出）且变为可选**；chatflow 必须以 answer 节点收尾
3. **Dify 已原生支持 human-input（人工介入）节点**：暂停等待用户填表单/审批后继续，sourceHandle = user_actions[].id 或 `__timeout`（超时分支）——R2 的 ask-user 设计直接对齐它，DSL 兼容无缝
4. Dify 官方已把图执行引擎抽成独立库 **langgenius/graphon**（Python）——可作执行语义的参考实现对照
5. 变量语法确认：模板插值 `{{#node_id.var#}}`（正则已核实）+ 结构化引用（字符串数组 selector）+ 四个虚拟前缀 `sys`/`env`/`conversation`/`rag`
6. Iteration：默认串行，并行上限 parallel_nums=10；错误模式三选一 terminated / continue-on-error（失败位填 null）/ remove-abnormal-output——我们的 Rust 引擎并行度不受 10 限制（上限抬高的点）
7. Loop：循环变量跨轮持久、三种终止机制（break_conditions / loop_count / loop-end 节点）
8. 节点错误策略枚举：fail-branch / default-value（R9 推荐已与事实对齐）
9. 节点全集 29 种（含 3 触发器、iteration-start/loop-start/loop-end 伪节点、RAG pipeline 专用 datasource/knowledge-index）

主要来源（存档）：
- https://github.com/langgenius/graphon/blob/main/src/graphon/enums.py （节点全集与执行分类权威来源）
- https://github.com/langgenius/dify/blob/main/api/constants/dsl_version.py （DSL 0.7.0）
- https://github.com/langgenius/dify/blob/main/api/services/app_dsl_service.py （DSL 导入导出结构）
- https://github.com/langgenius/graphon/blob/main/src/graphon/variables/template_resolution.py （变量正则）
- https://docs.dify.ai/en/cloud/use-dify/nodes/human-input （人工介入节点）
- https://docs.dify.ai/en/cloud/use-dify/nodes/iteration 、https://docs.dify.ai/en/cloud/use-dify/nodes/loop
- https://docs.dify.ai/en/self-host/use-dify/build/workflow-chatflow.md （workflow vs chatflow）

---

## Round 2 结论（2026-11-25）

### 已定决策 D11–D23

| # | 决策 | 来源 |
|---|---|---|
| D11 | 联动 = 继承快照（只读）+ 显式共享层；禁止 invocation 平级互读写；**子工作流本期做**（节点类型 `sub-workflow`，父派生子 invocation、结果回传） | R1 |
| D12 | human-input：①异步事件唤醒 + 控制面可封装长轮询造同步体感 ②单节点+模式字段（审批/表单/混合），**对齐 Dify human-input 语义**（user_actions 动作分支 + `__timeout` 超时分支）③挂起快照落库、重启后唤醒有效 | R2 |
| D13 | 引擎接口 = **异步提交 + SSE 事件流**（节点开始/完成/失败、LLM 增量、等待输入、终态），事件落库支持 Last-Event-ID 回放；不做同步挂起 | R3 |
| D14 | DSL 基线 0.7.0；不支持节点**占位导入**；workflow + chatflow 双类型支持 | R4 |
| D15 | 变量系统：兼容 `{{#node.var#}}` 模板（内部转结构化引用）；弱类型 + 可选 schema 标注 + **导入时静态校验引用可解析**；大对象**引用传递**（对象存储 uri） | R5 |
| D16 | Code 节点 = JS 进程内嵌池 + **Python 外部沙箱进程池**；Python 环境方案技术答复已给出（见附录），**待用户确认后定案** | R6 ⏳ |
| D17 | 安全：内网 + 服务级 API Key/JWT（预留 mTLS）；SSRF 黑白名单必做；Capability 密钥 AES-GCM 加密落库 | R7 |
| D18 | token 明细每节点每次 LLM 调用必做；**只记录 token 数 + model/provider，不做成本换算**（价格表后置） | R8 |
| D19 | 节点错误处理**完全对齐 Dify**：节点级重试次数/间隔 + fail-branch / default-value | R9 |
| D20 | 引入 **Conversation（会话）**实体；会话变量存引擎 DB（仅 assigner 可写）；生命周期归控制面——也是 invocation 间合法联动通道 | R10 |
| D21 | 触发器归**控制面**：调度/webhook/权限审计在 dsh，引擎只暴露统一 API 触发入口 | R11 |
| D22 | 生态节点分级：tool+api→**映射 Capability**；tool+mcp→引擎实现 MCP 客户端（本期）；tool+builtin→常用子集按需、其余占位；knowledge-retrieval→基础版**经 HTTP 能力对接企业知识库**（外部知识库同样 HTTP 对接）；agent→基础工具循环本期、复杂策略后置 | R12 |
| D23 | 【新需求】**密钥占位符机制**：引擎不持有真实密钥（xtoken 等），配置中传占位符，HTTP 请求**出网关时替换**——细节见 Round 3 S1 | R12 |

### 附录：R6 技术答复 — Rust 调用 Python 沙箱可行性与环境方案（待确认）

**结论：可以做。推荐"外部 Python worker 池"，不要把 CPython 嵌进引擎进程。**

路线对比：
1. **PyO3 进程内嵌 CPython**：三个致命问题——GIL 使所有 Python 执行串行化（多并发 code 节点互相排队）；C 扩展段错误直接带崩引擎进程（隔离归零）；部署耦合系统 Python。→ 否决，只适合单机工具。
2. **外部 worker 池（推荐）**：引擎维护 N 个 Python worker 进程；每次 code 执行派发到池中的 fresh 子进程（预载模板进程可把冷启动压到毫秒级）；入参/出参 = stdio/UDS 上的长度前缀 JSON；超时/超内存直接 kill 进程，隔离天然成立。与 Dify 官方 dify-sandbox 思路一致（受限 subprocess 沙箱）。

**Python 环境问题四层解决：**
1. **运行时自带**：用 python-build-standalone（uv 背后的独立发行版 Python）随引擎分发，不依赖客户机系统 Python/版本——解决"客户环境没有 Python 或版本不对"
2. **依赖预装**：部署时用 uv（本身就是 Rust 写的）按锁文件 `uv sync` 建 venv 预装企业常用包；**运行期禁止安装包**（安全 + 可复现）
3. **包白名单**：默认标准库 + 预装列表（对齐 Dify sandbox 受限包做法），新增包走部署变更而非运行时装
4. **资源加固**：Linux 下 seccomp/cgroups；整个 worker 池跑容器作为企业安全档位（R6 选项里已预留）；Windows 用 Job Object 限资源
5. **兼容 shim**：Dify code 节点签名 `def main(...)` 且入参/返回要求 JSON 可序列化 → worker 预置 shim 自动转换，DSL 导入零改动

➡️ 确认此方案则 D16 定案。

---

## Round 3 — 第三轮前沿问题（2026-11-25）

### ❓ S1 - 密钥占位符机制细化（D23 展开）

① **语法**：提议 `{{secret.name}}`（双大括号、无井号，与变量模板 `{{#node.var#}}` 视觉与解析均不冲突）；引擎对占位符**原样透传不解析**（引擎不知道真值），仅做格式合法性校验
② **允许出现的位置**：Capability 的 url/headers/auth、Dify http-request 节点配置、环境变量值；body 里是否允许？（允许则标注泄漏风险）
③ **egress 路由**：(a) 引擎**所有外部 HTTP 强制经网关**（egress proxy 指向网关），占位符必然被替换，SSRF 防护也在网关收口 (b) 仅含占位符的请求走网关，其余直连
④ **替换失败语义**：网关发现未配置的 secret → (a) 拒绝请求并回传错误（防占位符原样泄漏出网）(b) 原样发出

➡️ 推荐答案：① `{{secret.name}}`　② url/headers/auth/环境变量允许，body 默认允许但文档标注风险　③ (a) 强制经网关　④ (a) 拒绝+报错

**用户回答：** 同意推荐

---

### ❓ S2 - knowledge-retrieval 节点建模

保留 DSL 节点类型 `knowledge-retrieval`（DSL 兼容），执行 = 调用注册的 **Capability**（企业知识库 HTTP API）；多个知识库 = 各注册一个 Capability，节点配置选择或聚合；输入输出 JSON schema 企业内约定（query/top_k/… → docs 数组）。

➡️ 推荐答案：专用节点类型 + 后端 Capability，与 D22 一致。

**用户回答：** 同意推荐 + 【追加要求】兼容DSL的节点，控制面应该设计一套逻辑用于切换成我们自己的（细化见 Round 4 T2）

---

### ❓ S3 - http-request 与 http-tool 并存

Dify 的 `http-request`（URL/method 内联在节点配置）**保留原语义**（DSL 兼容）；`http-tool`（引用 Capability）为企业扩展节点；两者并存，SSRF 防护对两者都生效。

➡️ 推荐答案：并存。

**用户回答：** 同意推荐

---

### ❓ S4 - 难点节点映射确认

① **template-transform**：Dify 用 Jinja2 → Rust 用 minijinja（Jinja2 兼容实现，高覆盖率但非 100%，边界特性降级并在文档记录）
② **document-extractor**：内置基础格式（txt/md/csv/pdf 文本层），复杂格式（docx/pptx/扫描件 OCR）→ 建议走 Capability（企业文档解析服务）
③ **agent 节点**：基础版 = LLM + 工具循环（工具来自 Capability/MCP/builtin），策略对齐 function-calling 优先，ReACT 等模板后置

➡️ 推荐答案：如上分级。

**用户回答：** 同意推荐

---

### ❓ S5 - 入口 API 面 + SSE 事件 schema 提案

API 面：`POST /invocations`（提交）、`POST /invocations/{id}/cancel`、`/pause`、`/resume`、`/inputs`（human-input 提交）、`GET /invocations/{id}`（状态）、`GET /invocations/{id}/events`（SSE，Last-Event-ID 回放）。
事件类型：invocation_started / node_started / node_completed / node_failed / llm_delta（LLM 增量）/ waiting_input / input_submitted / invocation_finished / invocation_failed / invocation_cancelled / error；每事件含 invocation_id、seq（序号）、timestamp、node_id?、payload。

➡️ 推荐答案：确认方向，schema 细节由我出接口文档。

**用户回答：** 同意推荐

---

### ❓ S6 - 调度与并行语义

① DAG 调度：事件驱动，节点入度归零即调度；未选中分支的下游标记 skipped；variable-aggregator 取非 skipped 分支的输出
② iteration 并行：默认 parallel_nums=10（保持 Dify 兼容），**上限解锁为配置值**（我们不受 Dify 的 10 限制——抬高上限的点）
③ loop：对齐 Dify（循环变量跨轮持久 / break_conditions / loop_count / loop-end）
④ 子工作流嵌套深度上限：默认 5（防环，可配置）

➡️ 推荐答案：①③④ 如上，② 为明确改进。

**用户回答：** 同意推荐

---

## Round 3 结论（2026-11-25）

### 已定决策 D24–D29

| # | 决策 | 来源 |
|---|---|---|
| D24 | 密钥占位符：`{{secret.name}}`（引擎透传不解析、仅格式校验）；允许 url/headers/auth/环境变量，body 允许但标注风险；**所有外部 HTTP 强制经网关**；替换失败 = 拒绝+报错（ADR-0008） | S1 |
| D25 | knowledge-retrieval = 专用节点类型 + Capability 后端；【追加】**节点绑定切换机制**：兼容 DSL 的节点控制面可切换为自研实现——粒度待 Round 4 T2 | S2 |
| D26 | http-request（Dify 原语义）与 http-tool（Capability 引用）**并存**，SSRF 防护两者生效 | S3 |
| D27 | template-transform = minijinja（边界特性降级记录）；document-extractor 内置基础格式 + 复杂格式走 Capability；agent = 基础工具循环、function-calling 策略优先 | S4 |
| D28 | 入口 API 面 + 11 类 SSE 事件 schema 方向确认；schema 细节文档由设计方交付 | S5 |
| D29 | 调度：事件驱动 DAG（入度归零即调度、未选分支下游 skipped、variable-aggregator 取非 skipped 输出）；iteration parallel_nums 默认 10 但**上限解锁为配置值**；loop 对齐 Dify；子工作流嵌套深度默认 5 | S6 |

⏳ 遗留待确认：**D16**（R6 的 Python worker 池方案技术答复已给出，用户尚未明确确认）→ Round 4 T1

---

## Round 4 — 第四轮前沿问题（2026-11-25，预计收尾轮）

### ❓ T1 - D16 定案确认

上一轮 R6 的 Python 沙箱方案（外部 worker 池 + python-build-standalone 运行时自带 + uv 依赖预装 + 运行期禁装包 + 白名单 + seccomp/cgroups，容器池作为企业安全档位）技术答复已给出但未收到明确确认。

➡️ 推荐答案：确认定案。

**用户回答：** 同意推荐

---

### ❓ T2 - 节点绑定切换机制（S2 追加需求 D25 细化）

"兼容 DSL 的节点，控制面可切换成我们自己的"。提议定义为 **Node Binding（节点绑定）**机制：
① **粒度**：(a) 按节点类型全局切换（如全部 knowledge-retrieval → 企业 KB Capability）(b) 按节点实例切换（导入/编辑时替换单个节点实现）(c) **类型默认 + 实例覆盖**（类型级绑定作为导入默认，单个节点可覆盖）
② **契约**：绑定实现必须实现原节点类型的输入输出接口契约（否则 DSL 兼容承诺破坏）；切换的只是"实现"，不是"语义"
③ **存储与生效**：绑定关系存控制面（dsh）配置，随工作流定义下发引擎；引擎按绑定路由实现——默认 = Dify 语义忠实实现，绑定后 = 自研实现（如 Capability 后端、企业内部服务）

➡️ 推荐答案：①(c) ②强契约 ③如上。

**用户回答：** 同意推荐

---

### ❓ T3 - 需求补漏 + 交付物确认

① **需求补漏**：你最初说"目前我能想到就那么多"——设计冻结前最后一次补漏：还有没有未列出的需求？
② **交付物**：设计树清空后，我计划产出四份文档：
- A. **Dify 29 节点完整映射表**（每节点：DSL type / 执行语义 / 本引擎实现方式 / 绑定点 / 本期范围[全量|基础|占位]）
- B. **数据库 schema 骨架**（invocations / node_runs / 变量快照 / workflows DSL 版本 / capabilities / secrets / token_usage / conversations / conversation_variables / events）
- C. **入口 API + SSE 事件 schema 接口文档**
- D. **实施里程碑划分**（Rust crate 结构 + 阶段计划）
要全部，还是先要其中某些？

➡️ 推荐答案：② 四份全出，顺序 A→B→C→D；① 由你补充。

**用户回答：** 同意推荐

---

## Round 4 结论 — 设计树清空，达成共识（2026-11-26）

### 已定决策 D30–D32

| # | 决策 | 来源 |
|---|---|---|
| D30 | **D16 定案**：Code 节点 = JS 进程内嵌池（rquickjs）+ Python 外部 worker 池（python-build-standalone 自带运行时 + uv 依赖预装 + 运行期禁装包 + 白名单 + seccomp/cgroups；容器池=企业安全档位） | T1 |
| D31 | **Node Binding 定案**：① 类型默认绑定 + 节点实例覆盖 ② 绑定实现必须满足原节点类型输入输出契约（换实现不换语义）③ 绑定存控制面、随工作流定义下发引擎路由（ADR-0009） | T2 |
| D32 | 需求补漏 = 无新增需求；交付物 A/B/C/D 四份全出 | T3 |

### 交付物清单（本会话内产出）

| 文档 | 路径 |
|---|---|
| A. Dify 29 节点完整映射表 | docs/dify-node-mapping.md |
| B. 数据库 schema 骨架 | docs/db-schema.md |
| C. 入口 API + SSE 事件 schema | docs/api-spec.md |
| D. 实施里程碑（crate 结构 + 阶段计划） | docs/milestones.md |

### Grill 会话结束状态

- 设计树所有分支已访问完毕，前沿为空，无静默假设
- 决策总账：D1–D32（Round 1: D1–D10 / Round 2: D11–D23 / Round 3: D24–D29 / Round 4: D30–D32）
- ADR 清单：0001–0009
- 遗留开放项：无设计级未决项；schema/事件字段级细节在交付文档中给出初稿，实现期可迭代

---

## Round 5 — 复审重开（2026-11-26）

> 复审起因：Round 4 收尾写的「设计树所有分支已访问完毕，前沿为空」经不起对照。把 D1–D32 逐条对着 db-schema / api-spec / dify-node-mapping / milestones / GLOSSARY 核对后，发现 6 处文档互相矛盾、若干处「拍了板但没落到 schema/API 里」、1 条分支从未展开（dsh 自己的 workflow 怎么进引擎）。设计树重开，编号续为 U。
> 本轮落盘：GLOSSARY.md；ADR-0001~0005、0008、0009 修订；新增 ADR-0010~0013；新增 docs/open-issues.md。四份交付文档只改了直接矛盾的两行（api-spec 的 /resume、映射表的版本门槛），其余待设计树再次清空后统一修订。

### 复审定位（事实，非问题）

| # | 发现 | 位置（复审时） |
|---|---|---|
| 1 | 矛盾：Node Binding 存哪——D31 ③ 与 ADR-0009 说控制面，api-spec 与 db-schema 是引擎库 | grill-record D31；api-spec.md L24–25；db-schema.md L148–159 |
| 2 | 矛盾：引擎是否持有密钥——D23 说不持有，D17 ③ 与 ADR-0008 保留静态模式 | grill-record D23；ADR-0008 |
| 3 | 矛盾：联动通道——D11「只读快照 + 共享层」被 D20 可写会话变量推翻，共享层从未落地 | grill-record D11 / D20 |
| 4 | 矛盾：导入校验失败是报错还是进报告——R5 与 api-spec 422，对映射表 §8.3 | api-spec.md L63；dify-node-mapping.md L75 |
| 5 | 矛盾：/resume 与自动续跑重叠，/pause 是交付稿新增 | api-spec.md L14 |
| 6 | 矛盾：Http-Tool 不是引用 Capability 的唯一节点类型 | GLOSSARY.md L36；dify-node-mapping.md L27 |
| 7 | 缺口：定义只能来自 Dify DSL——dsl_yaml NOT NULL，只有导入入口 | db-schema.md L23 |
| 8 | 缺口：模型引用没有落脚点——Provider 配置在 schema/API 里无位置 | dify-node-mapping.md L23 |
| 9 | 后果未摆明：只收 0.7.0 会拒掉 Dify 1.16 之前的全部导出（依据 dify node.md 的调研，待复核） | dify-node-mapping.md L73 |
| 10 | 缺口：流式增量逐条落库 | db-schema.md L86–95 |
| 11 | 缺口：容器内节点执行无法区分 | db-schema.md L63 |
| 12 | 说法过于乐观：换队列即可多实例 | db-schema.md L186 |

### ❓ U1 - **引擎的「原生工作流定义」是什么**

现在定义工作流的入口只有 `POST /workflows/import`（Dify YAML），`workflow_versions.dsl_yaml` 是 `NOT NULL`，`graph_ir` 写的是「导入产物」。也就是四份文档都默认工作流来自 Dify。

可你的起点是 dsh 现有的 workflow，而 `http-tool`、`sub-workflow` 这类自有节点，ADR-0002 自己也说 Dify DSL 表达不了。dsh 画布画的、带自有节点的工作流，今天没有任何途径进引擎，这个概念在 GLOSSARY 里也没有名字。

(a) **Dify DSL 为底 + 扩展**：原生定义就是 Dify DSL 0.7.0 的结构。Dify 节点原样合法，自有节点类型注册进引擎的节点类型表，扩展字段放保留命名空间。Dify 导入就是归一化、校验、出报告，无损
(b) 独立 IR：引擎自有 schema，Dify DSL 和 dsh 画布各写一个编译器
(c) 维持现状

➡️ 推荐答案：(a)。D4 已经让 Dify 的结构成了你必须支持的子集，平行再造一套只有成本。`graph_ir` 降为每个版本的派生编译产物，不再是定义的来源；GLOSSARY 增补 **Workflow Definition（工作流定义）**。

**用户回答：** A ✅ [已定] → **D33**

---

### ❓ U2 - **配置类数据归谁存（修正一处矛盾）**

D31 和 ADR-0009 写的是绑定存控制面、随定义下发引擎。但 `api-spec.md` 里是引擎的 `PUT /workflows/{id}/bindings`，`db-schema.md` 里 `node_bindings` 表建在引擎库。工作流定义本身的处理是「写入口是控制面，数据存引擎库」，绑定没跟上。

(a) **引擎库单一存储**：执行时要用到的配置（工作流定义与版本、Node Binding、Capability、Provider 配置、静态密钥、会话变量）只存引擎库，控制面经引擎的管理 API 读写。控制面库只留自己的业务元数据（权限、目录、触发器日程）
(b) 控制面为源，每次调用随请求下发，引擎不存配置
(c) 两边都存

➡️ 推荐答案：(a)，并把 D31 ③ 和 ADR-0009 的措辞改成 (a)。

**用户回答：** 同意推荐 ✅ [已定] → **D34**

---

### ❓ U3 - **Provider 配置与模型引用（缺一个实体）**

Dify DSL 里模型只写 provider 和模型名，没有 endpoint 和密钥。D5 说 Provider「配置驱动」，但这个配置放哪、谁管，schema 和 API 里都没有。知识检索的 `dataset_ids`、插件工具引用同样悬空。

① Provider 配置的形态：(a) 部署期静态 YAML　(b) **受管实体**：入库 + 管理 API（增删改、连通性测试），密钥字段用 `{{secret.*}}`
② 导入时的模型映射：DSL 里每个 model 引用按映射规则解析，默认按 provider/model 名匹配已配置的 Provider，导入请求可带覆盖规则（如 `gpt-4o* → qwen-max`）；解析不了的进导入报告，节点不可运行直到配上

➡️ 推荐答案：①(b) ② 如上。存哪遵循 U2。

**用户回答：** 同意推荐 ✅ [已定] → **D35**

---

### ❓ U4 - **重启续跑：被杀时正在跑的节点怎么办**

D8 承诺重启续跑，schema §7 写的是扫 `node_runs` + `node_snapshots`。但节点如果在 kill 时跑到一半——比如 HTTP 请求已发出、对方已落库、只是响应没回来——它的输出就没有记录。重跑等于重复副作用，不重跑等于卡死。记录里没有这个决定。

(a) **至少一次**：在途节点重跑，沿用原执行的标识，并把它作为幂等键带在 HTTP 出站请求上，让后端能去重；契约里写明「崩溃恢复时节点可能执行不止一次」
(b) 在途节点置为「未知」并挂起，等人工裁决（重试 / 补录输出 / 取消）
(c) 节点级标注幂等性，默认 (a)，标了非幂等的走 (b)

➡️ 推荐答案：(a) 作为本期契约，状态机给「未知」留位子，等真有资金类流程再上 (c)。另外确认：重启后运行中的 Invocation 自动续跑（M1 验收是这么写的），不等人工调 `/resume`。

**用户回答：** 同意推荐，建立这几个方案的配置项，先实现A ✅ [已定] → **D36**

---

### ❓ U5 - **事件落库的粒度**

D13 和 ADR-0004 写的是事件同步落库，清单里有 `llm_delta` 和 `answer_delta`，`events` 表一条事件一行。那就是每个 token 增量写一次 MySQL：一路 1k token 的回答几百到上千行，100 路并发流是每秒几千到上万条插入，还卡在流式输出的热路径上。「单机 <1k 并发」会先死在这张表上，回放一堆 token 碎片也没有意义。

(a) 维持
(b) **分级**：结构性事件（Invocation 与节点的开始、完成、失败、等待输入、终态）同步落库，是审计和回放的依据。流式增量只在内存里广播、按 50–100ms 合并，不逐条落库；断线重连的客户端从 `node_completed` 或 `invocation_finished` 里拿完整文本
(c) 增量写 Redis Stream，带 TTL

➡️ 推荐答案：(b)。相应改 D13、D28、ADR-0004 的措辞：回放保证覆盖结构性事件。

**用户回答：** 不能每个token都写入吧，要按照节点执行后写入，你看看是不是没你说的问题

**回应：** 是，逐 token 写入的问题就此消除，等价于推荐的 (b)。补一处：节点「开始」也要落一行，而且先落库再执行——D36 里预留的另两种在途策略要靠它判断哪些节点在途。每节点多一行，量级不变。 ✅ [已定] → **D37**

---

### ❓ U6 - **容器里的节点执行没有身份**

`node_runs` 的标识是 (invocation_id, node_id, attempt)，`waiting_inputs` 主键是 (invocation_id, node_id)，快照按 `after_node_id`，SSE 事件只带 `node_id`。但 D29 定了并行迭代和循环：同一个 node_id 在并行迭代里同时跑 N 份，在循环里每轮跑一次。

于是哪条记录是哪个元素的、画布上的 node_started 属于第几轮、循环里的 human-input 等待记录被下一轮覆盖、重启时容器内进度怎么还原，全都分不出来。

(a) 不管，容器外能跑就行
(b) 引入 **Node Run（节点执行）**：一个节点在某一个作用域里的一次执行，迭代的一个元素、循环的一轮各算一次，重试算同一次。每次执行一个 `node_run_id`；容器本身也是一次 Node Run，子节点挂在它下面。事件信封加 `node_run_id`，变量解析、快照、等待记录都按它定位

➡️ 推荐答案：(b)。U4 的幂等键也用它。

**用户回答：** 同意推荐 ✅ [已定] → **D38**

---

### ❓ U7 - **会话：三件没定的事**

① **对话历史谁拥有**。LLM 节点的 `memory` 需要之前几轮的问答。映射表写「从 Conversation 注入历史轮次」，但引擎只存了会话和会话变量，没存消息；控制面的聊天界面有自己的消息表。
(a) 控制面每次提交时把 `history` 带进来，引擎对对话内容无状态　(b) 引擎从同一会话历史 Invocation 的 inputs 和 answer 输出里自己推导
➡️ 推荐答案：(a)，与「引擎只管执行」一致，避免两边各存一份消息。

② **同一会话并发**。用户连发两条或重试时，两个 Invocation 同时读写会话变量，后写覆盖先写。
(a) 引擎按会话排队，前一个到终态后一个才启动，`waiting_input` 期间也占着　(b) 直接拒绝第二个（409），控制面自己处理　(c) 允许并发，后写覆盖
➡️ 推荐答案：(a)。

③ **D11 的联动通道**。当时接受了「继承快照（只读）+ 显式共享层」，但「租户级共享层」之后没有任何表、API 或节点，而「只读快照、写不回」又被 D20 的可写会话变量推翻了。收窄为四条：输入、环境变量（只读）、会话变量（实时的）、子工作流返回值。租户级共享存储本期不做。
➡️ 推荐答案：是。

**用户回答：** 对话历史在dify的设计里时，显示注入的（通过会话变量，环境变量引入），工作流目前的定位是dsh的agent看得到的一个工具，拥有对话历史用处不大。 同一个会话并发，同意你的推荐。 D11联动通道，同意你的推荐

**解读：** ① 引擎不拥有对话历史，不存消息 → **D39**；同时记下新事实 **F1**（工作流目前的定位是 dsh 的 agent 可见的工具）。是否为导入的 chatflow 提供 history 入参，取决于 Dify memory 的真实机制，见 **F4** 与 Round 6 V3。② → **D40** ③ → **D41**

---

### ❓ U8 - **DSL 版本门槛**

映射表 §8.1 是 `version != "0.7.0"` 就拒绝。上一轮的调研报告写的是 0.7.0 在 Dify 1.16 才出现，1.15 及更早导出的是 0.6.0。企业环境的 Dify 普遍滞后，所以大部分真实的存量导出文件会在门口被拒，和 D4「存量迁移是硬卖点」直接冲突。R4 你同意过「历史版本不承诺」，但当时的推荐没把这个后果摆出来。

(a) 维持只收 0.7.0
(b) **接受版本区间 + 导入期归一化**：只拒绝比支持范围更新的文件；旧文件先升级（字段改名、旧的 app mode 判为不支持），再按 0.7.0 校验

➡️ 推荐答案：(b)。区间的下限和每个版本的差异已派子代理去查 Dify 源码的版本史（Dify 自己的导入器带升级逻辑，可以对照），结果回来后补充。

**用户回答：** 同意推荐 ✅ [已定] → **D42**（区间下限 [事实待查]，F2）

---

### ❓ U9 - **网关与密钥：强制到什么程度**

D23 写「引擎不持有真实密钥」，ADR-0008 和 D17 又保留了静态模式（AES-GCM 落库，引擎持有）。如果所有外部 HTTP 都强制过网关，静态模式就是多余的第二套实现。「强制」也没有无网关场景的说法（本地开发、POC、没有网关的客户）。

① 密钥引用只有一种语法 `{{secret.name}}`，由**部署档位**决定出口实现：`gateway`（占位符原样转发，企业默认）或 `local`（引擎用自己的密钥库解析并自己做 SSRF 校验）。「每个 Capability 选模式」取消
② 一个我查不到的事实：你们的网关现在能替换 **body** 里的占位符吗（JSON、multipart、流式）？不能的话 S1 ② 的「body 允许」要收回
③ MCP 本期只接远程传输（Streamable HTTP / SSE），同样走出口层；不支持本机 stdio（要在引擎宿主机起进程、密钥走环境变量，直接绕开网关）

➡️ 推荐答案：①③ 同意，② 请如实告知。

**用户回答：** 写一份open-issue，记录2，待补充，其他同意推荐

**处理：** ①③ ✅ [已定] → **D43**；② 记入 docs/open-issues.md 的 **OI-001** [事实待查]。

---

### ❓ U10 - **挂起语义**

① `/pause`、`/resume` 和 `paused` 状态是交付稿加的。你在 Q8 说的「暂停/恢复」指的是 ask-user 类节点，那是 `waiting_input` + `/inputs`。两套挂起机制现在重叠。
(a) 本期砍掉 `/pause`，唯一的挂起机制是 human-input，重启续跑自动（U4），状态机留扩展位　(b) 保留操作员暂停，那需要你说清场景（运维冻结？单步调试？）和在途节点、超时怎么处理
➡️ 推荐答案：(a)，除非你现在就有场景。

② **审批人**。schema 里 `waiting_inputs` 的注释写着用于「待我审批」查询，但没有审批人字段。
(a) 引擎不认识审批人，控制面自己记工单　(b) 引擎存一个可选的 `assignees`（不透明的 user/role 引用），列表 API 支持按它筛；权限校验仍在控制面
➡️ 推荐答案：(b)，一列加一个筛选条件的事，省得控制面再存一份等待状态。

**用户回答：** 第一个问题不砍掉吧，超时提醒/暂停/等用户输入。超时时间可以配置。第二个问题，同意推荐

**解读：** ① 保留 `/pause`，选的是 (b)，场景为「超时提醒 / 暂停 / 等用户输入」，超时时间可配。「超时提醒」的确切语义与暂停的细节待锐化 → Round 6 V1 ⏳ ② ✅ [已定] → **D45**

---

### ❓ U11 - **容量与多实例**

① 超过 1k 并发时：
(a) 内存队列无界，延迟悄悄变长　(b) **准入控制**：运行中上限（可配）+ 有界队列，超了回 429；再加一个可选的租户级并发上限，防止一个租户的批任务饿死其他人；加权公平调度以后再说
➡️ 推荐答案：(b)

② schema §7 和 D10 说多实例「换个队列实现就行」。实际不是：真正的阻塞是 SSE 扇出（客户端连在 A，Invocation 跑在 B）、唤醒和取消请求落错实例、崩溃后的接管租约。
(a) 本期明确只支持单实例，只保证三条便宜的约束：Invocation 状态不只活在内存、广播是落库事件之上的优化、队列任务按 invocation_id 路由。多实例之前单独做一轮设计　(b) 现在就把多实例设计进来
➡️ 推荐答案：(a)，并把 D10 的措辞改实。

**用户回答：** 都同意推荐。 ✅ [已定] → **D46、D47**

---

## Round 5 结论（2026-11-26）

### 新事实

- **F1**（用户陈述，U7）：工作流目前的定位是 dsh 的 agent 看得到的一个工具；对引擎而言拥有对话历史用处不大。
- **F2** [事实待查]：Dify DSL 版本史与各版本差异（子代理核查中），决定 D42 的区间下限。
- **F3** [事实待查]：dsh 现有工作流定义模型。dsh 源码不在 E:/github、E:/project 下，路径待用户提供。
- **F4** [事实待查]：Dify LLM 节点 memory 的历史来源。我方理解是 Dify 从该会话已存的消息里取最近若干轮，不经会话变量或环境变量；与用户在 U7① 的说法有出入，子代理核查中。

### 已定决策 D33–D47

| # | 决策 | 来源 |
|---|---|---|
| D33 | **Workflow Definition = Dify DSL 0.7.0 结构为底 + 扩展命名空间**；Dify 导入 = 归一化 + 校验 + 报告，无损；`graph_ir` 是每个版本的派生编译产物，不是定义的来源（ADR-0010） | U1 |
| D34 | **执行配置单一存储于引擎库**：定义与版本、Node Binding、Capability、Provider 配置、密钥库、会话变量只存引擎库，控制面经引擎管理 API 读写；控制面库只留业务元数据。D31 ③ 与 ADR-0009 的「存控制面」作废（ADR-0013） | U2 |
| D35 | **Provider 配置 = 受管实体**（入库 + 管理 API，密钥字段用 `{{secret.*}}`）；**导入期模型映射**：按规则解析 DSL 的 model 引用，默认同名匹配，可带覆盖规则，解析不了的进导入报告 | U3 |
| D36 | **重启续跑 = 至少一次**：在途节点重跑并沿用原执行标识，该标识作为幂等键随 HTTP 出站请求下发；在途策略做成配置项（重跑 / 挂起待裁决 / 按节点标注），本期只实现「重跑」；重启后运行中的 Invocation 自动续跑（ADR-0011） | U4 |
| D37 | **事件只在节点边界落库**：结构性事件（节点的开始、完成、失败、跳过、等待输入、终态）同步落库；流式增量不落库，内存广播 + 合并；回放保证范围 = 结构性事件（ADR-0012） | U5 |
| D38 | **Node Run 身份**：每次节点执行一个 `node_run_id`，容器本身也是一次 Node Run；事件、快照、等待记录、变量解析都按它定位 | U6 |
| D39 | **引擎不拥有对话历史**：不存消息，不做记忆注入；需要上下文由调用方经输入传入 | U7① |
| D40 | **会话内排队**：同一 Conversation 的 Invocation 串行，前一个到终态后一个才启动，`waiting_input` 期间也占着（排队的范围待 Round 6 V2 确认） | U7② |
| D41 | **联动通道收窄为四条**：输入、环境变量（只读）、会话变量（实时）、子工作流返回值；「继承快照」机制与租户级共享层不做 | U7③ |
| D42 | **DSL 版本区间 + 导入期归一化**：只拒绝高于支持上限的文件；旧版本先升级再按 0.7.0 校验。区间下限待 F2 | U8 |
| D43 | **密钥引用单一语法 `{{secret.name}}` + 部署档位出口**（`gateway` 企业默认 / `local` 无网关环境），取消「按 Capability 选模式」；**MCP 本期只接远程传输**，同样走出口层，不支持 stdio；网关对 body 的替换能力见 OI-001（ADR-0008） | U9 |
| D44 | **保留 `/pause`**：场景为超时提醒 / 暂停 / 等用户输入，超时时间可配。「超时提醒」的语义与暂停细节待 Round 6 V1 | U10① ⏳ |
| D45 | **审批人**：`waiting_inputs` 存可选的 `assignees`（不透明的 user/role 引用），列表 API 支持按它筛；权限校验仍在控制面 | U10② |
| D46 | **准入控制**：运行中上限（可配）+ 有界队列，超限回 429；可选的租户级并发上限；加权公平调度后置 | U11① |
| D47 | **本期只支持单实例**，撤回「换队列即可多实例」；只保证三条约束：Invocation 状态不只活在内存、广播是落库事件之上的优化、队列任务按 invocation_id 路由；多实例前单独设计 | U11② |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D8 | D36、D44 | 续跑补上在途节点语义（至少一次）；暂停保留 |
| D10 | D47 | 撤回「换队列即可多实例」，本期只支持单实例 |
| D11 | D41 | 「继承快照」与租户级共享层不做，联动通道收窄为四条 |
| D13、D28 | D37 | 事件落库范围收窄为结构性事件 |
| D14 | D42 | 版本策略由只收 0.7.0 改为区间 + 归一化 |
| D17 ③、D23、D24 | D43 | 密钥单一语法 + 部署档位；body 是否允许待 OI-001 |
| D31 ③ | D34 | 绑定存引擎库 |

ADR-0002 中「自有扩展节点无法在 Dify DSL 中表达」这条理由同样作废（ADR-0010），其结论（不承诺反向导出）保留。

### 派生结论（由已定决策直接推出）

1. 节点开始事件先落库再执行：D36 预留的另两种在途策略依赖在途检测，落点由 D37 提供。
2. 流式增量只在 SSE 连接存活期间可得，重连的客户端从 `node_completed` / `invocation_finished` 取完整文本（D37）。

### 默认处理（随 Round 6 向用户展示，用户未提异议，已按生效处理；清单见 Round 6）

- 取消向子工作流传播；子 Invocation 的截止时间不超过父 Invocation 的剩余时间。
- 幂等键命中返回同一个 Invocation，不论其状态；幂等键保留期可配，到期清理。
- 子工作流重跑不重复派生：子 Invocation 以父节点执行的标识作幂等键。
- 日志只输出结构化日志（带 invocation_id、node_run_id），引擎不存日志；追踪用 OpenTelemetry，可选。
- token_usage 另存厂商原始 usage（JSON），prompt / completion / total 三列照旧。
- 状态查询支持长轮询：`GET /invocations/{id}?wait=true&timeout=…`，到终态或等待输入时返回。

### Round 6 前沿

1. **V1** 「超时提醒」的语义与暂停的细节（D44 待锐化）
2. **V2** Conversation 与 Workflow 的关系、术语「会话」的归属（D40 的排队范围、ADR-0007）
3. **V3** 导入的 chatflow 里 LLM 节点的 memory 怎么办（D39 × F4）
4. **V4** 导入结果的严重度分级与「可运行 / 发布」门槛
5. **V5** Node Binding 与共享配置的生效时机、哪些节点允许类型级绑定
6. **V6** dsh 接入与存量工作流迁移——[事实待查] F3，阻塞中

---

## Round 6 — 第六轮（2026-11-26）

> 本轮落盘：GLOSSARY.md（Conversation 改写，新增 Session、Workflow Version）；ADR-0007 修订；新增 ADR-0014；api-spec.md 的 422 一行按 D53 改正。

### ❓ V1 - **「超时提醒」指的是什么，暂停怎么定义**

你给的三个场景里，「等用户输入」和「暂停」我清楚，「超时提醒」有歧义。

① 超时提醒：
(a) **等用户输入的两级时限**：human-input 挂起后，到可配置的「提醒时限」引擎发一个提醒事件，控制面据此让 agent 再问用户一遍；到「超时时限」仍无响应，走 `__timeout` 分支。两个时限都在节点上配
(b) **超时不失败，改为暂停并告警**：Invocation 或节点超过可配置的时限时不判失败，转成暂停并发事件，等人来决定继续还是取消
(c) 两者都要

➡️ 推荐答案：(a)。如果 (b) 才是你的意思，它要做成调用级的可选项（超时后：失败 / 暂停），默认失败，因为 D19 定了错误语义对齐 Dify。

② 暂停的定义（既然保留 `/pause`）：在途节点跑完并落库后停下，不再调度新节点。暂停期间 Invocation 超时和 human-input 的时限都停止计时。重启后暂停的 Invocation 保持暂停，自动续跑只针对重启时正在运行的。`waiting_input` 状态下不接受 `/pause`。

➡️ 推荐答案：如上。

**用户回答：** 同意推荐，超时还需要补充限制整个工作流执行总时间的语义吧，设置两个数字

**解读：** ①② ✅ [已定] → **D48、D49**。追加需求「限制整个工作流执行总时间，设置两个数字」，「两个数字」有歧义 → Round 7 W1。

---

### ❓ V2 - **Conversation 和 Workflow 是什么关系**

你说的「工作流是 dsh agent 的工具」让这件事必须定下来。一个 dsh 会话里，agent 可能并行调好几个工作流。现在 `conversation_id` 由控制面分配，会话变量表里只有 conversation_id 和 name，没有 workflow。

① (a) **Conversation 绑定单一 Workflow**（Dify 的语义：一个会话属于一个应用）：控制面按「dsh 会话 × 工作流」分配 conversation_id，会话变量按工作流声明，DSL 里本来就是这样。于是 U7② 的排队范围是「同一会话里的同一个工作流」：agent 并行调不同工作流互不阻塞，同一个工作流并行多次才排队
(b) **Conversation 就是 dsh 会话**，跨工作流共享：会话变量要变成三维键；排队范围是整个会话，agent 并行调两个不同工作流会被串行化，一个在等用户输入另一个就卡住

➡️ 推荐答案：(a)。

② 细化：工作流没有声明会话变量时，同一会话里并行调它也不排队，因为没有共享状态可冲突。你上次同意的是无条件排队，这是在知道「工作流是 agent 工具」之后的收窄，不要就维持无条件。

➡️ 推荐答案：加上这条。

③ 术语：GLOSSARY 把 Conversation 记作「会话」并禁用 session，但 dsh 自己叫「会话」的是 agent 的 session，现在是两个东西。

➡️ 推荐答案：增加 **Session（dsh 会话）** 条目；Conversation 保留，注明一个 Session 下每个工作流一个 Conversation。

**用户回答：** dsh 会话可以调用多个工具，其中可以包括多个工作流。同意你的细化工作流没有声明会话变量时，同一会话里并行调它也不排队。

**解读：** 用户确认了前提（一个 dsh 会话可以调多个工具，其中可以有多个工作流）→ 新事实 **F5**。② ✅ [已定]。① 用户没有单独作答，但 ② 是在 (a) 之下定义的，按 (a) 推定 → **D50**。③ 用户未提，按采纳处理 → **D51**。两条均已向用户标明，可被推翻。

---

### ❓ V3 - **导入的 chatflow 里 LLM 节点的 memory 怎么办**

你在 U7 说 Dify 的对话历史是通过会话变量、环境变量显式注入的。据我所知不是这样：Dify 的 LLM 节点上有一个 memory 配置（指定哪个变量是用户问题，加历史窗口大小），打开后 Dify 自己从该会话已存的消息里取最近若干轮拼进 prompt，读的是消息表。子代理正在源码里确认，有出入我更正。

如果属实，导入的 chatflow 只要开了 memory，在我们的引擎里就丢掉多轮上下文，行为和 Dify 不一致。在引擎不拥有对话历史（D39）的前提下：

(a) **本期降级**：memory 不生效，导入报告给每个开了 memory 的 LLM 节点标一条「记忆未生效」；需要上下文的工作流由调用方通过 inputs 传
(b) **最小支持**：`POST /invocations` 接受可选的 `history`，引擎按节点的 memory 配置拼进 prompt。引擎只消费，不存

➡️ 推荐答案：(a)。(b) 留作以后的小扩展，引擎侧改动不大。

**用户回答：** 这里只有dify自己设置的工作流memory需要兼容吧，与dsh会话的memory是两码事，为什么不做呢

**解读：** 用户否决推荐的 (a)，要求做。澄清了 U7① 的原意：用处不大的是 dsh 会话的 memory，Dify 工作流自己配置的 memory 属于 DSL 兼容范围，必须做 → **D52**。D39 相应修订。历史轮次由谁提供 → Round 7 W2。

---

### ❓ V4 - **导入结果的严重度分级与「可运行」门槛**

现在三处说法不一（R5 导入即报错、映射表进报告、api-spec 422）。占位节点的下游引用必然解析不了，按 422 处理就和 D14 的占位导入冲突。加上 D35 的模型映射，导入完的工作流常常要人补配置才能跑。

① 三级严重度：
- **错误**，无法成为工作流（YAML 解析失败、图结构非法、版本高于支持上限、不支持的应用类型）：拒绝导入，422
- **阻断项**，能导入但不能运行（占位节点、引用不可解析、模型引用未映射）：进报告，版本停在草稿
- **提示**，可运行但有差异（minijinja 不兼容、memory 未生效、插件需人工处理）：进报告

② 生命周期：版本分 `draft` 和 `published`。控制面显式发布，发布条件是没有阻断项。只有已发布的版本能被调用，`POST /invocations` 不带版本号时取当前已发布的版本。

➡️ 推荐答案：①② 如上。

**用户回答：** 同意推荐 ✅ [已定] → **D53**（「提示」级示例里的「memory 未生效」因 D52 作废）

---

### ❓ V5 - **Node Binding 与共享配置什么时候生效**

D34 定了绑定存引擎库，没定什么时候生效。这决定了改一个类型级绑定会不会影响正在跑的 Invocation。

① **绑定在发布时冻结**：发布的版本里每个节点已经解析出具体实现，改了绑定要重新发布才生效。发布时引擎校验绑定目标满足该节点类型的输入输出契约。类型级绑定的「全局切换」等于批量重新发布
② **共享运行配置不冻结**（Provider 配置、Capability 定义、密钥）：调用开始时解析，并给这次 Invocation 留一份快照，挂起几天后恢复仍用快照
③ **类型级绑定只对「配置里不带目标地址」的节点类型开放**：knowledge-retrieval、document-extractor、code、template-transform、human-input。http-request 和 tool 每个节点各有自己的地址，只允许实例级

➡️ 推荐答案：①②③ 如上。原则一句话：定义随版本冻结，共享配置按调用起始时解析。

**用户回答：** 同意推荐 ✅ [已定] → **D54**（ADR-0014）

---

### 默认处理（随本轮展示，用户未提异议，已按生效处理）

- 取消会向子工作流传播；子 Invocation 的截止时间不超过父 Invocation 的剩余时间
- 幂等键命中就返回同一个 Invocation，不管它是什么状态，失败的也不会重跑；保留期可配，到期清理
- 父节点重跑时不重复派生子 Invocation，子 Invocation 用父节点执行的标识作幂等键
- 日志只输出结构化日志（带 invocation_id、node_run_id），引擎不存日志；追踪用 OpenTelemetry，可选
- token_usage 另存一份厂商原始 usage（JSON），prompt / completion / total 三列照旧
- 状态查询支持长轮询：`GET /invocations/{id}?wait=true&timeout=…`，到终态或等待输入时返回

---

## Round 6 结论（2026-11-26）

### 新事实

- **F5**（用户陈述，V2）：一个 dsh 会话可以调用多个工具，其中可以包括多个工作流。

### 已定决策 D48–D54

| # | 决策 | 来源 |
|---|---|---|
| D48 | **超时提醒 = human-input 的两级时限**：到「提醒时限」发提醒事件，控制面据此让 agent 再问用户；到「超时时限」走 `__timeout` 分支。两个时限都在节点上配。「超时改为暂停并告警」不做 | V1① |
| D49 | **暂停的定义**：在途节点跑完并落库后停下，不再调度新节点；暂停期间 Invocation 超时和 human-input 时限停止计时；重启后保持暂停，自动续跑只针对重启时正在运行的；`waiting_input` 状态下不接受 `/pause` | V1② |
| D50 | **Conversation 绑定单一 Workflow**：conversation_id 由控制面按「Session × Workflow」分配，会话变量按工作流声明。排队范围 = 同一会话里的同一个工作流，且仅当该工作流声明了会话变量（ADR-0007） | V2①② |
| D51 | **术语 Session（dsh 会话）**：与 Conversation 是两个概念，一个 Session 下每个工作流一个 Conversation | V2③ |
| D52 | **Dify 工作流自带的 memory 必须兼容**：它与 dsh 会话的 memory 是两回事。历史轮次的来源待 Round 7 W2 | V3 |
| D53 | **导入严重度三级 + 版本生命周期**：错误级拒绝导入（422）；阻断项进报告，版本停在草稿；提示级进报告。版本分 `draft` / `published`，控制面显式发布，发布条件是没有阻断项，只有已发布的版本能被调用 | V4 |
| D54 | **生效时机**：Node Binding 在发布时冻结并做契约校验，改绑定要重新发布；共享运行配置（Provider 配置、Capability 定义、密钥）在调用开始时解析并留快照；类型级绑定只对配置里不带目标地址的节点类型开放（ADR-0014） | V5 |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D39 | D52 | 「引擎不拥有对话历史」收窄为「引擎不拥有 dsh 会话的历史」；Dify 工作流自带的 memory 要兼容 |
| D40 | D50 | 无条件排队改为：仅当工作流声明了会话变量才排队，范围是同一会话里的同一个工作流 |

### 新增术语

- **Session（dsh 会话）**、**Workflow Version（工作流版本）**；**Conversation** 的定义改写为绑定单一 Workflow。

### Round 7 前沿

1. **W1** 整个工作流的总时限：「两个数字」指什么、计的是什么时间、超时后怎么办
2. **W2** Dify 工作流 memory 的历史轮次由谁提供（D52 待定）
3. **W3** File 的形态；引擎对 File 是否只读不写
4. **W4** 环境变量的归属与生效时机；secret 类型变量导入后如何取值
5. **W5** dsh 接入与存量迁移怎么收口（V6，F3 仍未提供）

### 事实状态

- **F2** ✅ 已查，结果见下。D42 的区间下限与「比 0.7.0 新的文件怎么办」交 Round 7 W6 定。
- **F4** ✅ 已查，结果见下。结论成立：Dify 的 LLM 记忆来自该会话已存的消息，不经会话变量或环境变量。
- **F3** dsh 现有工作流定义模型，路径未提供。收口方式交 Round 7 W5。
- **F6** [事实待查] 0.7.0 之后未升版本号的新增（`knowledge-retrieval-v2`、`agent-v2`、human-input v2、`site`、LLM `model_selector`）的确切形态；工作流总执行时长与步数的全局上限；除 LLM 外哪些节点带 memory；human-input 的 timeout 语义；`.ifpkg` 从哪个版本起成为默认导出。已派子代理。
- **OI-001** 网关能否替换 body 里的占位符。

### 核查结果：Dify DSL 版本史（F2，子代理，2026-11-26）

来源：Dify 与 graphon 源码（常量的提交历史、各次升版的改动、tag 之间的节点实体差异）。子代理标为 UNVERIFIED 的项在这里保留标注。

| DSL 版本 | 引入的 Dify 版本（日期） | 对应的变化 | 导入该版本的文件时要处理的 |
|---|---|---|---|
| 0.1.0 | 0.6.15（2024-07-23） | 首个信封结构 kind / version / app + workflow 或 model_config | 旧的 variable-assigner 聚合器（output_type: array）；if-else 的扁平 conditions + logical_operator；节点没有 version / retry_config / error_strategy；插件化之前的 provider id；没有 dependencies |
| 0.1.1 | 0.7.0（2024-08-14） | conversation_variables；assigner 节点 v1 | assigner v1 的 assigned_variable_selector / write_mode / input_variable_selector |
| 0.1.2 | 0.8.0（2024-09-10） | 并行执行 | 未升版本号就加入的：iteration-start、document-extractor、list-operator、file / file-list 入参、HTTP body 改成列表（0.10.0） |
| 0.1.3 | 0.11.0（2024-11-05） | 发布性改动 | iteration 的并行字段 is_parallel / parallel_nums / error_handle_mode |
| 0.1.4 | 0.13.0（2024-12-03） | 变量赋值器 v2 | assigner v2（version: "2"、items[]）；所有节点开始带 version: "1" |
| 0.1.5 | 0.14.2（2024-12-23）至 1.2.0 | 重试与节点错误处理 | 跨越 1.0.0 插件化分界，仅凭版本号无法区分前后；agent 节点；dataset_ids 可能是 AES 加密的 |
| 0.2.0 | 1.3.0（2025-04-23） | LLM 结构化输出 | 缺省的 structured_output_enabled / structured_output |
| 0.3.0 | 1.4.0（2025-05-15） | assigner 增加 remove-first / remove-last | 只需补默认值 |
| 0.3.1 | 1.7.0（2025-07-23） | 工具的 credential_id、tool_node_version | 只需补默认值 |
| 0.4.0 | 1.8.0（2025-08-27） | start 的 checkbox；参数类型改名；code 输出类型放宽；LLM 的 reasoning_format | 参数提取器的 bool 改 boolean、select 改 string |
| 0.5.0 | 1.10.0（2025-11-13） | 触发器节点 | trigger-webhook / trigger-schedule / trigger-plugin |
| 0.6.0 | 1.13.0（2026-02-11） | human-input 节点 | human-input |
| 0.7.0 | 1.16.0（2026-07-17）；1.20.0 与 main 仍是它 | Agent 包 DSL（agent、agent_packages、app mode agent） | 宽容解析 |

要点：

- **版本号和内容不可靠地对应。** 节点 schema 在小版本里变过，也有不升版本号就改的。0.7.0 之后没升版本号又加了 `site`、LLM 的 `model_selector`、human-input v2、`knowledge-retrieval-v2`、`agent-v2`，以及默认的 `.ifpkg` 导出（2026-09-20）。`.ifpkg` 是 zip（manifest `format: dify.app` + DSL + 资源），显式 YAML 导出仍然存在。
- **旧式应用**（chat、completion、agent-chat）在任何 DSL 版本下都是 model_config 而不是 graph。Dify 原样导入，转换是另一个动作，会生成新应用。
- **Dify 自己的导入器不拒绝任何旧版本**：版本号非法才失败；比当前新或主版本更旧就让用户确认；次版本更旧只给警告。它做的归一化很少：缺省 version / kind、去掉 viewport、解密 dataset_ids、0.1.5 及以下推导插件依赖。节点级的改写发生在图初始化时（variable-assigner 改写成 variable-aggregator、human-input 的接收人、旧 tool_configurations 挪进 tool_parameters）。导出时一律盖上当前版本号。
- **子代理的建议**：区间下限取 0.1.5 并辅以内容嗅探，不依赖单个字符串；无论版本号是多少，都按节点 `data.type` 对照引擎支持的类型来校验。

UNVERIFIED：0.1.1 / 0.1.3 / 0.1.5 三次升版的理由（按时间和发版说明推的）；旧的 output_type: array 现在是否还被处理；旧 if-else 的 sourceHandle 到 case_id 的映射；main 上的导入是否校验节点类型（只在 1.9.0 确认了不校验）；「输出」是否只是界面显示名；0.6.15 之前的导出格式；存量 DSL 的版本分布。

### 核查结果：End 节点与节点全集（子代理，2026-11-26）

- End 节点的 type 字符串在 graphon main（94109ce8，2026-11-25）和 Dify 1.9.0 里都是 `end`。不存在 `output` 类型。
- graphon 的 BuiltinNodeTypes 有 25 个成员，自首次提交未变：start、end、answer、llm、knowledge-retrieval、if-else、code、template-transform、question-classifier、http-request、tool、datasource、variable-aggregator、variable-assigner（遗留）、loop、loop-start、loop-end、iteration、iteration-start、parameter-extractor、assigner、document-extractor、list-operator、agent、human-input。Dify 1.9.0 也是 25 个，差别是 1.9.0 有 knowledge-index、没有 human-input。没有共同的字符串被改过。trigger-* 不在 graphon 里，在 `api/constants/trigger.py`。
- 对上一轮文档的更正已写入 dify node.md 的「勘误」和 dify-node-mapping.md。

### 核查结果：human-input 与 LLM memory（子代理，2026-11-26）

- **human-input 没有审批人模型**，谁拿到表单 token 谁就能提交（提交接口不要求登录，只有 IP 限流）。节点数据是 form_content、inputs、user_actions（自由定义的按钮）、timeout、delivery_methods；接收人只是投递目标。D45 的 assignees 是我们的扩展。
- **LLM 节点的 memory 配置**在 1.9.0 和 graphon main 上相同：`role_prefix`（user / assistant，可空）、`window`（enabled、size）、`query_prompt_template`（可空），整个 memory 也可以为空。
- **运行时的历史来源**：从变量池读 `sys.conversation_id`，没有它（workflow 应用）就没有记忆；按 conversation_id 查 `messages` 表，最新的在前，query 作用户轮、answer 作助手轮，文件来自 message_files。`window.size` 只在 `window.enabled` 时生效，上限 500 条消息、2000 tokens。
- **更正**：V3 里把 memory 配置说成「指定哪个变量是用户问题，加历史窗口大小」，形状说错了；「历史来自已存的消息」这个结论经核实成立。

### Round 7 追加前沿

6. **W6** DSL 版本区间的下限；比 0.7.0 新的文件怎么办；`.ifpkg` 是否接受（依据 F2）

---

## Round 7 — 第七轮（2026-11-26）

> 本轮中途收到子代理的 Dify 版本史核查结果（见 Round 6 结论的「核查结果」），据此追加了 W6。

### ❓ W1 - **整个工作流的总时限：两个数字指什么**

你说要补上限制整个工作流执行总时间的语义，设两个数字。这四个字我有两种读法，下面这套设计两种都覆盖。

① 两个数字是工作流级别的**提醒时限**和**超时时限**，和 human-input 节点上那对一样，提醒时限可以不配。如果你指的是「节点上一个数、整个工作流一个数」，把提醒时限留空就是那个模型
② 超时时限只累计**运行中的时间**：从开始运行起算，等用户输入和暂停期间不计时，它们各有自己的时限；排队等起跑的时间也不算
③ 到超时时限：Invocation 判失败（错误码 TIMEOUT），在途节点和子工作流一起取消，没有分支可走。配置优先级是引擎默认值、工作流定义里的值、提交调用时的 `timeout_ms`（已有的入参）和 `remind_after_ms`，后者覆盖前者

➡️ 推荐答案：①②③ 如上。

**用户回答：** 一个是用户输入的时间上限，一个是工作流执行的时间上限

**解读：** 「两个数字」= human-input 的等待上限 + 整个工作流的执行时间上限。工作流级别只有一个数字，① 里提的「工作流级提醒时限」不要。②③ 用户未作答，转入默认处理（见本轮结论）。 ✅ [已定] → **D55**

---

### ❓ W2 - **Dify 工作流 memory 的历史从哪来**

你说得对，这个该做。U7 那句我读成了「工作流也不需要记忆」，是我读错了。剩下的只有一个问题：节点要的历史轮次由谁提供。

(a) **引擎自己记**：每个 Conversation 里，引擎给每次 Invocation 记一个轮次（用户的查询加 answer 节点的回答文本）。节点开了 memory，引擎就按窗口大小取这个 Conversation 最近几轮拼进 prompt。这是 Dify 自己的做法，V2 又把 Conversation 定成了引擎按工作流持有的实体，所以不需要新接口
(b) **控制面传入**：`POST /invocations` 带 `history`，引擎只消费不存

➡️ 推荐答案：(a)。(b) 要控制面替每个工作流工具另记一份轮次，而引擎手里本来就有这些数据。

**用户回答：** 同意推荐， 这个不是自动功能吧，是用户在dify启用的时候才有的吧

**解读：** (a) ✅ [已定] → **D56**。用户的问题是「这是不是自动功能」：不是。只有节点上配置了 memory 才注入历史；节点没配就完全不碰历史。「记轮次」也不是一项单独的功能：轮次的查询就是这次调用的输入，回答就是 answer 节点的输出，本来就要存。

---

### ❓ W3 - **File 长什么样，引擎能不能写**

大对象走引用传递（D15）定了，但 File 本身没定义过。

① 形态是 `{name, mime_type, size, url}`，上传由控制面完成，引擎不碰上传
② 本期引擎对 File **只读不写**：文档提取和 LLM 视觉输入由引擎经出口层下载内容，图片内联成 base64，因为内网地址厂商拉不到；大小上限可配。HTTP 节点返回二进制、代码节点生成图片这类「工作流产出文件」本期不支持

➡️ 推荐答案：①② 如上。② 的前提是你们目前没有工作流产出文件的需求，有的话引擎得能写对象存储，请说。

**用户回答：** 有产出文件类的需求，另外我希望知道rust对office的文件（xlsx, pptx, docx和pdf)有没有编辑的库，同意你的推荐

**解读：** ① ✅ [已定] → **D57**。② 的前提被推翻：有工作流产出文件的需求，引擎必须能写文件 → Round 8 X2。另提出事实问题 **F7**：Rust 有没有编辑 xlsx / pptx / docx / pdf 的库（已派子代理）。

---

### ❓ W4 - **环境变量归谁、什么时候生效**

Dify DSL 的环境变量（`environment_variables`）在 grill 里没讨论过。它是应用级的，可以放 API key 这类 secret 类型的值，导出时会被置空。

① 声明和值都是工作流定义的一部分，随版本冻结，和 Dify 一样改了要重新发布
② 导入时值为空的 secret 类型变量自动转成 `{{secret.变量名}}`，真实值走密钥库或网关，导入报告里注明；非 secret 类型的空值留空，发布前由人补

➡️ 推荐答案：①② 如上。

**用户回答：** 同意推荐 ✅ [已定] → **D58**

---

### ❓ W5 - **dsh 接入与存量迁移怎么收口**

源码路径我问过两次，没拿到。引擎这边其实只需要定一条边界：引擎提供「用 Workflow Definition 新建版本、更新草稿、发布」的 API，不限于 Dify 导入；dsh 的画布格式怎么变成 Workflow Definition 是控制面的事。

(a) 本轮 grill 只定这条边界，dsh 侧的转换器和迁移策略等接入启动时另起一轮
(b) 现在给路径，这一轮把 dsh 画布格式查清楚、定完

➡️ 推荐答案：(a)。你两次没给路径，我理解为这轮不想看 dsh 源码；想换 (b) 随时说。

**用户回答：** 目前这台机器没有代码，我待会给GitHub地址，你上传到GitHub上，我在有代码的机器上拉取本仓库，所以需要记录grill过程，其他上下文也可以写入一份handoff文件或者memory文件

**解读：** F3 的原因明确了：这台机器上没有 dsh 源码。用户的计划：稍后提供 GitHub 地址，把本仓库推上去，在另一台有 dsh 源码的机器上拉取后继续。 ✅ [已定] → **D59**：本轮只定边界，dsh 侧的细节在有源码的机器上另起一轮。附带动作：grill 过程照常记录；新写 HANDOFF.md；等地址到位后推送。

---

### ❓ W6 - **版本区间的下限，比 0.7.0 新的文件怎么办**（本轮中途追加）

D42 定了接受区间加归一化，下限当时待查，现在事实有了。

① 下限：
(a) `0.1.5`，Dify 0.14.2 起：2024-12 至今的导出都收。这个版本号横跨 1.0.0 插件化的分界，要靠内容判断文件在分界前还是分界后（看有没有 `dependencies`、provider id 的形状）
(b) `0.2.0`，Dify 1.3.0 起：2025-04 至今。从这里往上全是增量变化，归一化最干净，但 1.3.0 之前的导出全拒
(c) `0.1.0`：全收。要多做 assigner v1 的转换、旧的 HTTP body 形态、旧的聚合器，而且这些全是插件化之前的文件

② 比 0.7.0 新的文件：D42 定的是拒绝。
(a) 维持拒绝
(b) 改为照常导入，报告置顶一条警告。未知的节点类型本来就会变成占位节点，未知字段原样保留；版本号挡不住真正的不兼容，节点类型校验才挡得住

③ `.ifpkg`：导入接口一并接受，解包取里面的 DSL，资源文件忽略。

➡️ 推荐答案：①(a) ②(b) ③是。

**用户回答：**（未作答）[待决] → Round 8 重问（X1）

---

### 默认处理

**随 Round 7 展示，用户未提异议，已按生效处理：**

- 草稿的编辑方式：一次导入或新建产生一个草稿版本，草稿可以原地改，发布后不再变；要改就从已发布版本派生新草稿
- 触发器指向 workflow_id，触发时取当前已发布版本；在途的 Invocation 用它开始时的版本跑完
- Capability 的输入 schema 和节点配置对不上时，调用在开始时就失败，并给出明确的错误
- conversation_id 第一次出现时引擎建 Conversation，会话变量按当时版本的声明初始化默认值，之后版本新增的变量在第一次被引用时补默认值；`sys.dialogue_count` 取该 Conversation 里已有的调用数
- 知识检索节点导入后默认是阻断项，发布前必须绑定后端
- DSL 里的 `features`（开场白、推荐问题、语音等）是应用层特性，归控制面，引擎忽略
- 每个 Invocation 的节点执行总数有上限，防失控；做成引擎配置项，默认值实现期对照 Dify 定
- Dify 的旧式应用（chat、completion、agent-chat，DSL 里是 `model_config` 而不是 `graph`）判为错误级，提示先在 Dify 里转换成 Chatflow 再导出

**W1 的 ②③ 用户未作答，按下面的默认处理（随 Round 8 展示）：**

- 工作流执行时间上限只累计运行中的时间：从开始运行起算，等用户输入和暂停期间不计时，它们各有自己的时限；排队等起跑的时间也不算
- 到上限：Invocation 判失败（错误码 TIMEOUT），在途节点和子工作流一起取消，没有分支可走
- 配置优先级：引擎默认值 → 工作流定义里的值 → 提交调用时的 `timeout_ms`
- human-input 节点上的提醒时限（V1 定的）保留为可选项；工作流级别不设提醒时限
- 队列先进先出，不做优先级；以后需要时按提交来源加

---

## Round 7 结论（2026-11-26）

### 事实状态变化

- **F3** 原因明确：这台机器上没有 dsh 源码。用户将提供 GitHub 地址，在有源码的机器上拉取本仓库后继续（见 D59 与 HANDOFF.md）。
- **F7** [事实待查] Rust 有没有编辑 xlsx / pptx / docx / pdf 的库。已派子代理。

### 已定决策 D55–D59

| # | 决策 | 来源 |
|---|---|---|
| D55 | **两个时间上限**：human-input 的等待上限（节点上配）+ 整个工作流的执行时间上限（一个数字）。工作流级别不设提醒时限 | W1 |
| D56 | **memory 由引擎提供**：引擎为每个 Conversation 里的每次 Invocation 保留一个轮次（用户的查询 + answer 节点的回答文本），节点配置了 memory 就按窗口取最近几轮拼进 prompt。不是自动功能：节点没配 memory 就完全不碰历史 | W2 |
| D57 | **File 的形态**：`{name, mime_type, size, url}`，引用传递。引擎能否写文件待 Round 8 X2 | W3① |
| D58 | **环境变量属于工作流定义，随版本冻结**；导入时值为空的 secret 类型变量自动转成 `{{secret.变量名}}`，真实值走密钥库或网关，导入报告里注明；非 secret 类型的空值留空，发布前由人补 | W4 |
| D59 | **dsh 接入的收口**：引擎只认 Workflow Definition（以及作为其来源之一的 Dify 导入），提供新建版本、更新草稿、发布的 API；dsh 画布格式到 Workflow Definition 的转换归控制面。dsh 侧的转换器与迁移策略在有源码的机器上另起一轮 | W5 |

### 新增术语

- **Turn（对话轮次）**、**Memory（记忆）**、**File（文件）**、**Environment Variable（环境变量）**。

### Round 8 前沿

1. **X1** W6 重问：版本区间的下限；比 0.7.0 新的文件怎么办；`.ifpkg` 是否接受
2. **X2** 工作流产出文件：文件存储怎么做、本期哪些节点能产出文件
3. **X3** Office 文档的生成和编辑放在哪一层（待 F7）
4. **X4** 运行数据的留存与清理

---

## Round 8 — 第八轮（2026-11-26）

> 本轮 X1–X4 均已答。X1 是上一轮追加的 W6。

### ❓ X1 - **Dify 版本区间的下限，比 0.7.0 新的文件怎么办**

这是上一轮中途追加的 W6。

① 下限：
(a) `0.1.5`，Dify 0.14.2 起：2024-12 至今的导出都收。这个版本号横跨 1.0.0 插件化的分界，要靠内容判断文件在分界前还是分界后
(b) `0.2.0`，Dify 1.3.0 起：2025-04 至今。从这里往上全是增量变化，归一化最干净，但更早的导出全拒
(c) `0.1.0`：全收。要多做 assigner v1 的转换、旧的 HTTP body 形态、旧的聚合器

② 比 0.7.0 新的文件：D42 定的是拒绝。
(a) 维持拒绝
(b) 改为照常导入，报告置顶一条警告。未知的节点类型本来就会变成占位节点，未知字段原样保留

③ `.ifpkg`：导入接口一并接受，解包取里面的 DSL，资源文件忽略。

➡️ 推荐答案：①(a) ②(b) ③是。

**用户回答：** 版本不用考虑太多，就最近的也没问题了，同意你的推荐

**解读：** ①(a) ②(b) ③是 ✅ [已定] → **D60**。用户补充：版本跨度不是硬需求，只支持较新的版本也可以接受。所以实现时若 0.1.5 的归一化成本偏高，把下限抬到 0.2.0 不需要重新决策。

---

### ❓ X2 - **工作流产出文件怎么做**

你说有产出文件的需求，W3 里「引擎对文件只读」的前提就不成立了。

① **文件存储**：引擎加一个可插拔的文件存储接口（对象存储，本地开发用本地目录），和出口层一样按部署档位配置。节点产出的文件一律经它写入，按租户和 Invocation 分前缀
② 引用沿用 W3 定的形态。引用里的地址是存储的内部地址，对外访问由控制面换签名链接
③ 本期哪些节点能产出文件：
(a) HTTP 节点拿到二进制响应
(b) code 节点写出文件，worker 把文件交回引擎
(c) 子工作流把文件回传给父级
(d) 模型产出，比如生图、生视频

➡️ 推荐答案：①② 如上；③ 本期做 (a)(b)(c)，(d) 取决于厂商接口会不会返回文件，后置。

**用户回答：** 同意推荐，可以包括图，word, ppt, excel的文件

**解读：** ✅ [已定] → **D61**。用户补充了产出文件的类型：图片、Word、PPT、Excel。对 ③(d) 的影响见下方默认处理。

---

### ❓ X3 - **Office 文档的生成和编辑放哪一层**

(a) **code 节点**：Python worker 预装上面那些成熟库，引擎不为 Office 写专用节点
(b) **Rust 内置节点**：只做库撑得住的，比如 xlsx 的读写、docx 和 pptx 的模板填充
(c) **Capability**：外部的文档服务。Office 转 PDF 据我所知在 Rust 里得靠 LibreOffice 这类外部引擎，复杂编辑也一样

➡️ 推荐答案：先 (a)+(c)。(b) 等 (a) 里出现重复的模式再沉淀成节点。子代理核实后如果 Rust 库比我记的成熟，再调整。

另请用户举例：产出 Office 文档的典型场景（模板填充、报表导出、格式转换、改既有文件），用来判断 (b) 值不值得做。

**用户回答：** office 文档生成和编辑形成一个功能节点， 可以考虑C#的库，只要跨语言调用没问题，成本不太高，调用，维护，部署

**解读：** 用户没有在 (a)(b)(c) 里选，而是给了新方向：Office 文档的生成和编辑要收拢成**一个功能节点**；实现可以考虑 C# 的库，条件是跨语言调用可行，且调用、维护、部署的成本都不高 ✅ [已定] → **D62**。节点的形态、文件怎么传、操作范围 → Round 9；实现语言与调用方式待事实（F7、F8）。用户没有举场景，只列了文件类型。

---

### ❓ X4 - **运行数据留存多久**

每次 Invocation 会留下调用记录、节点执行记录、事件、变量快照、轮次、token 明细，现在还有产出的文件。全部永久保留的话，库会无限长大。

(a) 不清理
(b) **分类留存**：变量快照在 Invocation 进入终态后删除，续跑不再需要它；调用记录、节点执行记录、事件、轮次按可配的保留期清理，默认 180 天；token 明细留 2 年；产出文件默认留 7 天；都支持按租户覆盖
(c) 在 (b) 之上加一个审计档：租户可以开启「保留变量快照直到调用记录过期」

➡️ 推荐答案：(b)。D7 选每节点快照时把「审计」列为了理由，但事后排查需要的是每个节点的输入输出，那在节点执行记录里，照常保留；变量快照只是续跑用的。如果确实需要事后看到每个节点的完整变量上下文，就选 (c)。

**用户回答：** 同意推荐 ✅ [已定] → **D63**

---

### 默认处理（随本轮展示，用户未提异议，已按生效处理）

W1 只答了「两个数字」是哪两个，下面几条是默认处理：

- 工作流执行时间上限只累计运行中的时间：从开始运行起算，等用户输入和暂停期间不计时，它们各有自己的时限；排队等起跑的时间也不算
- 到上限时 Invocation 判失败（错误码 TIMEOUT），在途节点和子工作流一起取消
- 配置优先级是引擎默认值、工作流定义里的值、提交调用时的 `timeout_ms`，后者覆盖前者
- human-input 节点上的提醒时限（V1 定的）保留为可选项，工作流级别不设提醒时限
- 队列先进先出，不做优先级

X2 用户列了图片，由此新增一条（随 Round 9 展示）：

- 如果「图」指的是模型生成的图，「生图」从后置改为本期做：厂商返回 URL 或 base64 时，引擎经出口层下载后写入文件存储。生视频仍后置。

---

## Round 8 结论（2026-11-26）

### 新事实与变化

- **F7** [事实待查] Rust 的 Office 编辑库。子代理在查。
- **F8** [事实待查] C# 的 Office 库（能力、许可证、维护状态），以及 Rust 调 C# 的方式与成本（独立服务、进程池、hostfxr 进程内、NativeAOT 动态库）。已派子代理。
- **OI-001** 仍待网关侧确认。

### 已定决策 D60–D63

| # | 决策 | 来源 |
|---|---|---|
| D60 | **Dify 版本策略**：接受 `0.1.5` 至 `0.7.0`，靠内容判断文件在插件化分界之前还是之后；比 0.7.0 新的文件照常导入并在报告置顶警告，不拒绝；导入接口一并接受 `.ifpkg`（解包取 DSL，资源忽略）。无论版本号是多少，都按节点 `data.type` 对照引擎支持的类型校验。下限不是硬需求，实现期可抬到 0.2.0 | X1 |
| D61 | **工作流可以产出文件**：引擎有一个可插拔的文件存储接口（对象存储，本地开发用本地目录），按部署档位配置；节点产出的文件一律经它写入，按租户和 Invocation 分前缀。File 的引用沿用 D57 的形态，地址是存储的内部地址，对外访问由控制面换签名链接。本期产出文件的入口：HTTP 节点的二进制响应、code 节点写出文件、子工作流回传。产出类型包括图片、Word、PPT、Excel | X2 |
| D62 | **Office 文档的生成和编辑收拢成一个功能节点**；实现可以考虑 C# 的库，前提是跨语言调用可行，且调用、维护、部署的成本都不高。节点的形态、文件怎么传、操作范围见 Round 9；实现语言与调用方式待 F7、F8 | X3 |
| D63 | **运行数据分类留存**：变量快照在 Invocation 进入终态后删除；调用记录、节点执行记录、事件、轮次默认留 180 天；token 明细留 2 年；产出文件默认留 7 天；均可按租户覆盖 | X4 |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D57 | D61 | File 从「引擎只读」改为引擎可写：有文件存储，节点可以产出文件 |

### Round 9 前沿

1. **Y1** 这个功能节点以什么形态出现：Capability + Http-Tool Node，还是专用节点类型
2. **Y2** 文件怎么在引擎和外部服务之间传：内联 base64、预签名地址，还是服务直连文件存储
3. **Y3** 文档节点本期支持哪些操作
4. **Y4** 实现语言与调用方式（C# 独立服务、Python worker、Rust 内置）——依赖 F7、F8

---

## Round 9 — 第九轮（2026-11-26）

> 初稿写于交接前。按约定一轮是答完才写入；这一轮当时先写，是因为用户要把仓库推到 GitHub、在另一台机器上接着做。Y1–Y3 已于 2026-11-26 在第 10 轮开头重提后作答，答案见各题下方；Y4 还没问。

### ❓ Y1 - **这个功能节点以什么形态出现**

第 10 轮开头重提时补充的新事实：现在的「Markdown 转 docx」是一个 Python 第三方工具，既能当会话工具，也能在图里当节点，已经接近 (a)，只是它是本机进程而不是 HTTP 服务。

D62 定了 Office 文档的生成和编辑收拢成一个功能节点。两种落法：

(a) **Capability + Http-Tool Node**：Office 文档服务是一个独立的 HTTP 服务，把它的接口注册成 Capability，工作流里用现有的 http-tool 节点从下拉框选它。引擎不用为它写代码，和 Round 0 说的「快速封装 HTTP 接口形成新功能节点」是同一件事；以后换实现只要改 Capability 指向的服务
(b) **专用节点类型**：契约固定（操作、模板文件、数据、输出文件），画布上更好用，文件类型的输入输出能在发布时校验；引擎要为这个节点类型写代码

➡️ 推荐答案：(a) 先做。(b) 等画布体验不够了再加，到时就是一个绑定到这个 Capability 的专用节点，和知识检索的做法一样。

**用户回答：** A ✅ [已定] → **D64**

---

### ❓ Y2 - **文件怎么在引擎和外部服务之间传**

第 10 轮开头重提时补充的新事实：dsh 现在完全靠本机绝对路径传文件，所以不管选哪个，dsh 这一侧都要由控制面搬运（见 Round 10 的默认处理）。

HTTP 节点产出文件、Office 服务读写文件，都会碰到同一个问题：文件怎么进出一次 HTTP 调用。

(a) **内联 base64**：文件内容放在 JSON 里。最简单，任何服务都能接；受大小限制
(b) **预签名地址**：引擎向文件存储申请有时效的下载和上传地址，服务用 HTTP 取输入、回传输出。不占请求体，适合大文件。要求存储支持预签名；本地目录的开发档位下，由引擎自己提供带令牌的上传下载端点
(c) **服务直连文件存储**：把存储的凭据交给服务。耦合存储实现，凭据扩散

➡️ 推荐答案：默认自动，小文件内联，超过可配的阈值（默认 10MB）用预签名地址；Capability 可以在定义里指定只用其中一种。

**用户回答：** 同意推荐 ✅ [已定] → **D65**

---

### ❓ Y3 - **文档节点本期支持哪些操作**

第 10 轮开头重提时补充的新事实：现在只有「从零生成」里的一种（Markdown 转 docx，学术模板）。这条归类被用户更正，见下方的回答。

(1) **模板填充**：给 docx、xlsx、pptx 模板和数据，输出文件
(2) **从零生成**：按结构化数据生成表格、文档、幻灯片
(3) **编辑既有文件**：在已有文档上做指定修改，比如替换单元格、追加段落或页、插入图片
(4) **格式转换**：Office 转 PDF 等，需要渲染引擎或商业库

➡️ 推荐答案：本期做 (1)(2)(3)，(4) 后置。

**用户回答：** 同意推荐，你说的md2docx是类似模板填充的样式

**解读：** 本期做 (1)(2)(3)，(4) 后置 ✅ [已定] → **D66**。用户更正了我对现有能力的归类：Markdown 转 docx 是套着学术模板生成的，属于 (1) 模板填充这一类，不是 (2) 从零生成。所以本期三类操作里已经有一类是现成的。

---

### ❓ Y4 - **实现语言与调用方式**（未问）

C# 独立服务、Python worker、Rust 内置之间怎么选，等 F7、F8 的事实到了再问。

---

## 核查结果：dsh 现有工作流实现（F3，2026-11-26）

在有 dsh 源码的机器上查清。对象是插件仓里名字带 dify 和 workflow 的四个插件，以及它们依赖的内核接口。完整事实和出处见根目录的 `dsh workflow.md`，这里只列影响设计的结论。没有改任何决策。

1. **dsh 现有的工作流定义是画布图，不是 Dify DSL 的子集。** `WorkflowGraph { nodes, edges, vars }`，22 种节点，没有子图，表达式是 JavaScript，条件、URL、入参、输出选择器都能写任意 JS。D33 的 Workflow Definition 读不了它，D59 说的控制面转换器必须真写一个。
2. **在用的节点以 Python 代码为主体。** 插件仓里 4 份在用工作流的快照共 79 个节点：code 30（全是 Python，约 127KB）、agent 13、ifelse 6、userinput 5、pipeline 4、knowledge 3、tool 3、vars 3、document 2、subflow 2、start 与 end 与 answer 共 8。本机开发库另有 14 份画布图，多用到 http 和 answer。`parallel`、`loop`、`var`、`template`、`vagg`、`phase`、`log` 没有用例。
3. **现有的 Dify 导入是有损的，方向是 Dify → 画布图。** `loop`、`question-classifier`、`parameter-extractor`、`list-operator`、`agent`、`human-input` 和触发器都变成不可执行的占位节点；LLM 节点的结构化输出和 memory 丢失；`template-transform` 的 Jinja2 被当成 JS 模板字符串。在 dsh 里导入过的 Dify 工作流，原始 DSL 已经无法从画布图还原。
4. **「发布」只是元数据，保存即生效。** 每次保存都生成新版本，执行永远取最新版；`status` 不参与执行，草稿只要不是 `hidden` 就被注册成会话工具。有「撤回发布」，没有版本冻结。`hidden`、`scope`、`projectId` 是控制面的业务元数据。
5. **三个触发入口，没有定时和 webhook。** 会话工具（每个非 hidden 的存档一个）；`POST /graph/trial-run`（画布已不调用）；`POST /graph/runs`（外部网关的入口，异步，靠轮询）。
6. **执行是「编译成一段 JS，放进 worker 线程的 vm 里跑」。** 重活经 RPC 回宿主；总超时默认且硬上限 900 秒；重启时运行中的一律置 failed，没有续跑；一个会话同时只允许一个含人工输入的运行。
7. **LLM 节点在会话内是完整的子 agent。** 走 `subagents.startActivation`，带着会话的 agent 设定，不是一次模型调用；不流式、不统计 token。宿主的模型路由都是 OpenAI 兼容协议，D5 的主干成立。
8. **并行的瓶颈有具体来源。** 宿主里工具默认独占执行，工作流工具都在此列；子 agent 的活跃上限默认是 8，LLM 节点又都是子 agent。
9. **人工输入有两套，互不相干。** `ask_user_question` 是内核工具，只有根 agent 能用；画布的 `userinput` 是 bridge 自己做的待答卡，分段执行加 checkpoint，前端轮询，没有超时。
10. **会话变量（dsh-vars）是 dsh 会话级、跨工作流、agent 可读的字符串桶。** 它和 Dify 按「会话 × 工作流」的 conversation_variables 不是一个东西。在用的工作流靠它把文件的绝对路径和整篇文本交给 agent。
11. **文件靠本机绝对路径传递。** 输入是路径或 URL 字符串；产物写在数据目录下；交付卡由宿主读本机路径；大结果写回执文件再把路径给 agent。引擎一旦不在同一台机器上，这几条都要重新设计。
12. **Office 文档现在的全部能力是 Markdown 转 docx**，实现是一个 Python 第三方工具，会话工具和图内节点共用。
13. **「节点上选后端」已经是现成做法。** knowledge 节点三选一（Dify 数据集接口、一个内部知识库平台的 OpenAPI、一个 MCP 服务），document 节点二选一。这是 Node Binding 实例级覆盖的先例。其中内部平台要用共享密钥现铸短期 JWT，不是 `{{secret.name}}` 的替换能表达的。
14. **企业版跑的是内核内部分支。** `appendEvent`、`declareDeliverables`、工具执行上下文的 `sessionId` 只在这份分支里有。工具调用的 `callId` 由模型提供方签发，可能是合成值或空串，不能当隔离键。
15. **dsh 在做会话级的 token 归属**（已设计未实施）：内核按工具、会话自身、压缩、skill 四类记账，插件每轮写 MySQL。LLM 调用移到引擎后，这部分用量 dsh 的会话账看不到。
16. **dsh 已在用 MySQL**：会话日志的持久化，以及 `dsh-custom-agent` 插件自建的表。

对已定决策的影响（都留给 Round 10，这里不改决策）：

| 决策 | 现状带来的问题 |
|---|---|
| D59（转换归控制面） | 转换器的输入是画布图。任意 JS 表达式在 Dify 的结构里没有对应物，要有去处 |
| D53、D54（发布冻结） | dsh 现在保存即生效、草稿可调用，接入后用户的工作方式会变。下线的语义仍没定，dsh 有「撤回发布」可参照 |
| D3（tenant / user） | dsh 只有登录账号，没有租户 |
| D50、D41（Conversation 绑单一工作流） | dsh-vars 是会话级、跨工作流的桶，agent 直接读；在用的工作流依赖它 |
| D57、D61、Y2（File 引用） | dsh 与引擎之间的文件怎么过没有设计：输入的上传、产物的下载、交付卡要本机路径 |
| D18（token 明细） | 引擎的用量和 dsh 的会话归属账是两本账，口径也不同 |
| D43、ADR-0008（密钥占位符） | 需要运行时计算的鉴权不在占位符的范围内 |
| D21（触发归控制面） | 外部网关那条路的契约要保持 |

这 8 条的去向见 Round 10 结论；还没解决的都记在 `docs/open-issues.md`。

---

## Round 9 结论（2026-11-26）

### 已定决策 D64–D66

| # | 决策 | 来源 |
|---|---|---|
| D64 | **Office 文档节点 = Capability + Http-Tool Node**：文档服务是独立的 HTTP 服务，接口注册成 Capability，工作流里用 http-tool 节点引用，引擎不为它写节点代码。专用节点类型等画布体验不够了再加，届时是一个绑定到这个 Capability 的专用节点 | Y1 |
| D65 | **文件在引擎与外部服务之间的传递**：默认自动，小文件内联，超过可配的阈值（默认 10MB）用预签名地址；Capability 可以在定义里指定只用其中一种。dsh 一侧由控制面搬运（见 Round 10 的默认处理） | Y2 |
| D66 | **文档节点本期支持的操作**：模板填充、从零生成、编辑既有文件；格式转换后置。现有的 Markdown 转 docx 属于模板填充 | Y3 |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D61 | D64、D65 | 文件的交换方式定为「内联或预签名地址」；Office 文档服务经 Capability 接入 |

### 遗留

- **Y4** 实现语言与调用方式：等 F7、F8 的事实。D62 里「实现可以考虑 C#」仍然有效，待 Y4 定。

---

## Round 10 — 第十轮：dsh 接入与存量迁移（2026-11-26）

> 依据：上面的核查结果「dsh 现有工作流实现（F3）」。开头重提了第 9 轮的 Y1–Y3，随后是 Z1–Z6。
> 作答情况：Z1、Z4、Z5、Z6 已答；Z2 用户决定到迁移的时候再指挥；Z3 用户第一次没看懂题目，第 11 轮举例重述后作答（D73）。

### ❓ Z1 - **画布图怎么进引擎**

① 路线：
(a) 控制面写一个转换器（画布图转成 Workflow Definition），画布照旧编辑画布图，同一个转换器用于存量迁移。这期间在引擎里导入的 Dify 工作流不经画布编辑。
(b) 画布改为直接编辑 Workflow Definition，画布图退役，存量图用同一个转换器一次性迁移。
(c) 引擎再实现一套画布图的执行器。

② 画布里的 JS 表达式放到哪：
(a) 引擎加一种扩展能力，字符串字段可以写 JS 表达式，由引擎在 JS 沙箱里求值。
(b) 引擎不加。转换器把导入器生成的那几种写法还原成 Dify 的结构化写法，还原不了的降级成一个 JS 代码节点；画布上照样允许用户写 JS。

➡️ 推荐：① 选 (a) 先做，(b) 是方向，等画布需要编辑 Dify 独有的节点时再做；(c) 不做，两套执行语义必然漂移。② 选 (b)，引擎只认 Dify 的结构加少数扩展节点，JS 只存在于代码节点。

**用户回答：** 同意推荐

**解读：** ① (a) 先做，(b) 是方向，(c) 不做；② (b) ✅ [已定] → **D67**（ADR-0015）

---

### ❓ Z2 - **存量迁移的范围**

插件仓里有 4 份在用工作流的快照，本机开发库里有 14 份画布图，每个存档留最近 10 个版本。生产库里到底有多少，我确认不了。

(a) 只迁「每个用户实际生效的那一份」的最新版本：转成草稿，附转换报告，确认后发布。版本历史、运行记录、调用记录、产物都不迁。
(b) 连版本历史一起迁。
(c) 不迁，在引擎里按原始 Dify DSL 重新导入。

➡️ 推荐 (a)。(c) 不行：其中三份导入后在画布上改过很多。细节按默认做法处理：子流程由按名字改为按 workflow_id，找不到的记为阻断项；shared 种子和用户的分叉副本各自迁成独立工作流，由人决定留哪个；现在能被调用的工作流迁完直接发布。

**用户回答：** 到时候我会指挥你迁移, 工作流存放在哪，怎么打包进入镜像

**解读：** 这一题没有在 (a)(b)(c) 里选，定的是做法：迁移到的时候由用户指挥。届时要定两件事：工作流（定义）存放在哪；怎么打包进镜像。对设计的约束：引擎里的工作流要能随部署包交付，不只是从画布上来。推荐答案里那几条细节处理（子流程改按 workflow_id、分叉副本各自迁、现在能调用的迁完直接发布）没有被采纳也没有被否，届时一起定。[待决] → `docs/open-issues.md` 的 OI-017

---

### ❓ Z3 - **会话变量留在控制面，还是并入引擎**

dsh 的会话变量是 dsh 会话级的字符串桶，跨工作流共享，agent 可以直接读写。在用的工作流靠它把产物的路径和整篇报告交给 agent。引擎里的 Conversation 是「会话 × 工作流」，两者对不上。

(a) dsh-vars 留在控制面不动。画布的 `vars` 节点转成引擎里调用控制面接口的节点，执行时带上会话引用。Dify 自己的会话变量仍归引擎，两套并存。
(b) 并入引擎：Conversation 改成按 dsh 会话划分。这要改 D50 和 D40 的排队规则。
(c) 不再支持，agent 只从工具返回值里拿结果。

➡️ 推荐 (a)。它是 dsh 自己的功能，不属于 Dify 兼容；挪进引擎要推翻 D50，还会让同一会话里的不同工作流互相排队。

**用户回答：** vars 为什么会导致同一个会话的不同工作流互相排队，这个知识dsh的session和工作流共享一些变量，这题目我没看懂

**解读：** 没有作答，题目写得绕。「这个知识…」我按「这个只是…」理解，即用户认为这件事只是 dsh 的 Session 和工作流之间共享一些变量——这正是 (a) 的出发点。重述后再问，见 Round 11 开头。[待答]

---

### ❓ Z4 - **`tenant_id` 取什么**

D3 定了每次调用必填 `tenant_id` 和 `user_id`。dsh 里只有登录账号，没有租户。

(a) `tenant_id` 用部署级的固定值，`user_id` 用登录账号的 id，以后真有多租户再加映射。
(b) 用项目当租户。

➡️ 推荐 (a)。项目会变，一个用户可以在多个项目里。

**用户回答：** 同意推荐，可以由一个环境变量写死

**解读：** (a)，并且 `tenant_id` 的值由一个环境变量配置 ✅ [已定] → **D68**

---

### ❓ Z5 - **已发布版本的下线**

交接文件里记着「下线的语义没讨论过」。dsh 现在有「撤回发布」，引擎只有草稿和已发布两态。

(a) 引擎加「下线」：下线后不接受新调用，在途的跑完为止，可以重新发布。控制面撤回时同时下线引擎里的版本并注销工具。
(b) 引擎不加，撤回只注销控制面的工具。

➡️ 推荐 (a)。成本是一个状态转换。

**用户回答：** A

**解读：** 选 (a) ✅ [已定] → **D69**（ADR-0014 修订）

---

### ❓ Z6 - **工作流消耗的 token 要不要计入 dsh 的会话账**

dsh 在做会话级的 token 归属，按工具、会话自身、压缩、skill 四类记账并写 MySQL。LLM 调用移到引擎后，工作流内部的消耗不会出现在这张表上。

(a) 计入：引擎在调用记录上给出汇总用量，dsh 的汇总表加一类「工作流」条目；引擎库里的明细照旧。为了让两本账对得上，`token_usage` 增加缓存读、缓存写、推理三个字段。
(b) 不计入，各记各的。

➡️ 推荐 (a)。少了工作流内部的消耗，「哪个工具花得多」会严重偏低；两边都还没实施，现在定接口最便宜。

**用户回答：** 要，这个先自己单独记账

**解读：** 工作流的 token 要有账，但先由引擎按 D18 自己单独记，暂不并入 dsh 的会话归属账；(a) 里「汇总表加一类条目」和「增加三个字段」本期都不做。「要」是指要有账（我的解读），「先」是指以后可能再并（我的解读）。✅ [已定] → **D70**

---

## Round 10 结论（2026-11-26）

### 已定决策 D67–D70

| # | 决策 | 来源 |
|---|---|---|
| D67 | **画布图经控制面的转换器进引擎**：转换器（画布图 → Workflow Definition）先做，画布照旧编辑画布图，同一个转换器用于存量迁移；画布改为直接编辑 Workflow Definition 是方向，等画布需要编辑 Dify 独有的节点时再做；引擎不实现画布图的执行器。**引擎不增加 JS 表达式扩展**：转换器把导入器生成过的写法还原成 Dify 的结构化写法，还原不了的降级成一个 JS 代码节点，在 JS 沙箱里执行；画布上仍然允许写 JS，由转换器负责降级（ADR-0015） | Z1 |
| D68 | **`tenant_id` 是部署级的固定值，由一个环境变量配置**；`user_id` 用登录账号的 id。以后真有多租户再加映射 | Z4 |
| D69 | **引擎支持「下线」**：已发布的版本可以下线，下线后不再接受新的调用，在途的跑完为止，可以重新发布。控制面的「撤回发布」同时下线引擎里的版本并注销工具（ADR-0014 修订） | Z5 |
| D70 | **工作流消耗的 token 先由引擎单独记账**：按 D18 记进 `token_usage`，暂不并入 dsh 的会话归属账。以后要不要并入再定 | Z6 |

### 未决

- **Z2 存量迁移**：到迁移的时候由用户指挥，届时定工作流存放在哪、怎么打包进镜像（OI-017）。
- **Z3 会话变量的归属**：题目重述后再答。

### 默认处理（随本轮展示，用户未提异议，已按生效处理）

1. **dsh 与引擎之间的文件由控制面搬运**：调用前把 file 型入参里的本机文件上传到引擎的文件存储；调用后把产出文件下载到 dsh 本机目录再声明交付卡，宿主接口不用改。两者同机或共享存储时这一步没有成本。
2. **Invocation 带控制面的会话引用 `session_ref`**，列表 API 可以按它筛。用途是重连或重启后把等待输入的调用重新呈现成卡片。
3. **`hidden`、`scope`、`projectId` 不进引擎**，仍是控制面的业务元数据。工具注册改为已发布且非 hidden 才注册；工具名的派生规则不变。
4. **LLM 节点改为引擎直连模型**（现有两条路由都是 OpenAI 兼容协议），密钥走密钥库或网关。结果会和现在不同，因为不再带着会话的 agent 设定。
5. **时间上限**：引擎默认的执行时间上限设为 1 小时（可配）；dsh 侧工具超时的上限不再写死 900 秒。
6. **外部网关的契约不变**（发起返回 jobId、轮询查状态、submit 作答），内部改调引擎。
7. **需要运行时计算的鉴权**（比如用共享密钥现铸 JWT）不进出口层的占位符机制；这类后端包成 HTTP 服务再注册为 Capability。
8. **引擎库和 dsh 的库分开**（可以同实例），互不建外键。工具调用的幂等键不用 dsh 的 `callId`，它可能是空串或重复。

D53 落地后的一处行为变化，接入时要向用户说明：画布保存只产生或更新草稿，要发布才生效，草稿不能被调用。dsh 现在是保存即生效。

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D3 | D68 | `tenant_id` 的取值具体化为「环境变量配置的部署级固定值」 |
| D18 | D70 | 不变；明确工作流的 token 和 dsh 的会话归属账是两本账，本期不合并 |
| D53、ADR-0014 | D69 | 已发布的版本可以下线 |
| D59 | D67 | 转换器先做；引擎不加 JS 表达式，由转换器降级 |
| D55 的默认处理 | 本轮默认处理第 5 条 | 引擎默认的执行时间上限是 1 小时 |
| D21 | 本轮默认处理第 6 条 | 外部网关的对外契约不变，改调引擎 |

### 新增术语

- **Unpublish（下线）**、**Session Variable（dsh 会话变量）**。后者是为了和 Conversation 的会话变量区分开，是我提的叫法，可以被推翻。

### Round 11 前沿

1. **Z3** 重述后再答。
2. **Y4** 实现语言与调用方式，等 F7、F8。
3. Capability 的响应适配放在哪：现有几个后端的响应都要经过一段适配才符合节点的输出契约（OI-005）。
4. 一个会话里同时有多个等人输入的调用时，dsh 怎么呈现（OI-006）。
5. 建立在运行记录上的会话工具需要哪些引擎接口（OI-007）。
6. 现有的 Python 第三方工具迁到引擎后以什么形态存在（OI-008）。

Round 11 起题目编号改用两个字母，从 AA 开始。

---

## Round 11 — 第十一轮（2026-11-26）

> 第 10 轮没答的 Z3 在这一轮开头重述，随后是 AA1、AA2。F7、F8 的核查结果在本轮中途到，据此把第 9 轮遗留的 Y4 也问了。
> 作答情况：AA1、AA2 已答；Z3 在第二次重述并举例之后作答（D73）；Y4 用户答「待定」，倾向 Python 服务先做。

### ❓ Z3 - **会话变量留在控制面，还是并入引擎**（重述）

先把名字分开。「会话变量」是引擎的：属于某个 Conversation，按工作流声明，只有 Dify 的赋值节点能写。dsh 自己另有一个变量桶（dsh-vars 插件），按 dsh 会话分，agent 和工作流都能读写，叫它「dsh 会话变量」。

问题本身：现在这个桶由 dsh 存。工作流搬进引擎后，画布上的 `vars` 节点在引擎里执行，它要写的这个桶放在哪？

(a) 还放在 dsh。引擎执行到这个节点时，调一下 dsh 的接口去写。
(b) 搬进引擎，变成引擎的会话变量。

「互相排队」的来历：D40 规定同一个 Conversation 里的调用排队，前一个跑完后一个才开始，为的是两个调用同时改同一个会话变量时不互相覆盖。范围很窄：同一个会话里的同一个工作流，而且它声明了会话变量。如果选 (b)，dsh 的桶是整个会话共用的，所以同一个会话里的所有工作流都落进同一个 Conversation。于是 agent 先调工作流 A，A 在等人输入；这时再调工作流 B，B 必须等 A 整个跑完才能开始。选 (a) 没有这个问题：A 和 B 在引擎里是两个 Conversation，互不等待，只是执行到 `vars` 节点时各自去写 dsh 的桶。

➡️ 推荐 (a)。

**用户回答：**（第一次重述时没有作答；那一轮只答了 AA1 和 AA2）

**第二次重述**（用户在题目上批注：「没看懂，举个例子」）：用一份在用的工作流举例。它先让用户逐项确认几个要素，每确认一项就把值记进 dsh 会话变量；检索完之后，再把产物的路径和报告的正文记进去；工作流结束以后，agent 调用的「修改报告」「生成文档」这些工具，是从这个变量桶里读到路径和正文的。变量桶现在由 dsh 保管。所以要定的是：工作流在引擎里执行以后，「记进变量桶」这一步写到哪。(a) 变量桶还由 dsh 保管，引擎执行到该节点时调 dsh 的接口，什么都没变；(b) 改由引擎保管，dsh 的变量插件和两个读写工具都要改，还要改三条已定的决策（D40 排队规则、D50 会话的划分、D56 记忆的归属）。

**更正：** 前两次把「互相排队」当作推荐 (a) 的主要理由，不妥。它只影响同一个 dsh 会话里两个都要用这个变量桶的工作流，而且那种情况下前一个跑完、后一个再跑，反而能防止后写的盖掉先写的。推荐 (a) 的主要理由是：它是 dsh 自己的功能，不属于 Dify 兼容，留在 dsh 不用改动任何已定的决策。

**用户回答：** A

**解读：** 选 (a)。✅ [已定] → **D73**

---

### ❓ AA1 - **各个后端的响应适配放在哪**

知识检索现在有三个后端，文档解析有两个。每个后端返回的形状都不一样，现有代码里各有一段适配逻辑把它们变成统一的形状，有的还要改字段名、处理分数的含义。文档服务走 Capability 已经定了（Y1），但 Capability 的定义里只有地址、鉴权、超时和输入输出 schema，没有「怎么变换响应」。

(a) 每个后端前面放一个很小的 HTTP 适配服务，Capability 指向它。
(b) Capability 的定义里带一段字段映射。
(c) 引擎为某个后端写原生实现，再用 Node Binding 指过去。

➡️ 推荐 (a)。引擎只说 HTTP，各后端的差异都挡在适配服务里；代价是每个后端多一个要部署的小服务。

**用户回答：** 同意推荐

**解读：** (a) ✅ [已定] → **D71**

---

### ❓ AA2 - **一个会话里同时有多个工作流在等人输入时，卡片怎么显示**

dsh 现在「一个会话同时只允许一个等人输入的运行」，输入框上方也只有一张卡。接入后引擎没有这个限制，agent 可以先发起 A（等人确认），再发起 B（也等人确认）。

(a) 控制面保留限制：会话里已经有一个等人输入的调用时，再发起含人工输入的工作流就被拒绝。
(b) 前端支持多张卡，各自提交。

➡️ 推荐 (a) 先做，和现在的行为一致。现在的待答卡本身还有欠账（只渲染第一个字段），先不扩。

**用户回答：** 同意推荐

**解读：** (a) ✅ [已定] → **D72**

---

### 默认处理（随本轮展示，用户未提异议，已按生效处理）

1. 现有的 Python 第三方工具（文档生成和两个检索工具）各自包成 HTTP 服务，注册成 Capability。它们现在的协议几乎就是一个 HTTP 接口的形状。
2. dsh 里那几个「对已完成运行的报告做修改」的会话工具，由引擎提供两样：按调用 id 取结果和节点输出；按会话和工作流取最近一次成功的调用。

---

### ❓ Y4 - **文档服务用什么语言实现** [待定]

第 9 轮遗留的题。D64 已经定了文档服务是独立的 HTTP 服务，引擎只发 HTTP，所以「Rust 怎么调 C#」不再是问题。剩下的是这个服务用什么写。依据是 F7、F8 和补查的 Python 事实（细节在 `office libs.md`）：

- Rust：xlsx 够用；docx 能读写，但做模板填充的库要么停更、要么只有一个版本；pptx 的库全在 0.x；没有库支持艺术字。
- C#：免费的 Open XML SDK 和 ClosedXML 很成熟，但偏底层，模板填充要自己拼 XML；改既有文件和转 PDF 能力最全的是商业套件。EPPlus、NPOI 的官方二进制、QuestPDF 商用要付费。
- Python：python-docx、python-pptx、openpyxl 都是 MIT，是这三种格式最常用的高层库，docxtpl 专门做 docx 的模板填充。本机已经装着前三个。只核实了版本和前三个的许可证。

(a) **C# 服务**：用免费的 Open XML SDK 加 ClosedXML，需要高保真编辑或转 PDF 时上商业套件。部署物是一个自包含的 .NET 服务。
(b) **Python 服务**：用 python-docx、python-pptx、openpyxl、docxtpl，转 PDF 靠 LibreOffice。现在的 Markdown 转 docx 工具本来就是 Python，可以作为起点。部署物是 Python 镜像加字体。
(c) **Rust 服务**：和引擎同一种语言，但库最弱，改既有文件和 pptx 要自己补很多。

➡️ 推荐 (b) 先做。三类操作在 Python 里都有现成的高层库，团队已经在用，起步最快。(a) 留作升级路线：服务的接口按 D64 固定下来，以后需要商业库的能力时，换一个 C# 的实现指向同一个 Capability，工作流不用改。(c) 不做。如果本来就有现成的 C# 文档服务，或者有 C# 的人手，就改推荐 (a)。

**用户回答：** 待定， 目前我倾向于B 到 A

**解读：** 没有定案。倾向和推荐一致：(b) Python 服务先做，以后需要时换成 (a) C# 服务。接口按 D64 固定下来，换实现不影响工作流。[待定] → OI-003

---

## Round 11 结论（2026-11-26）

### 已定决策 D71–D73

| # | 决策 | 来源 |
|---|---|---|
| D71 | **Capability 的响应适配放在后端前面的 HTTP 适配服务里**：后端的响应和节点的输出契约不一致时，在后端前面放一个小的适配服务，Capability 指向它；Capability 的定义里不做响应变换（ADR-0005 修订） | AA1 |
| D72 | **一个 dsh 会话里同时只有一个等人输入的调用**：这是控制面自己的限制，会话里已经有一个等人输入的调用时，再发起含人工输入的工作流就被拒绝；前端仍是一张卡。等真有「同时等多个」的场景再支持多张卡 | AA2 |
| D73 | **dsh 会话变量留在 dsh**：变量桶仍由 dsh 的会话变量插件保管，引擎不持有它。画布的 `vars` 节点经转换器变成调用 dsh 接口的 http-tool 节点，执行时带上会话引用；agent 的读写工具不变。Dify 自己的会话变量仍按 D20、D50 归引擎，两套并存 | Z3 |

### 派生结论

- dsh 要给引擎开一个带服务令牌的会话变量读写接口。现有的路由要求登录态，引擎的回调没有用户会话（OI-063）。
- 会话变量节点要能引用 `session_ref` 来寻址，所以 `session_ref` 要在变量池里能被节点引用。

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D9、ADR-0005 | D71 | Capability 指向的可以是后端前面的适配服务 |

D40、D41、D50、D56 都不因 D73 改动：dsh 会话变量是引擎之外的通道。

### 未决

- **Y4**：待定，用户倾向 (b) Python 服务先做、以后 (a) C# 服务。
- **Z2**：到迁移的时候由用户指挥（OI-017）。

### 下一轮前沿

1. Y4（待定）。
2. 文档服务的接口契约，以及对选定的库做保真度验证（OI-059）。
3. 复审：把 D1–D73 逐条对着库表和接口文档核一遍，再判断设计树是不是走完了。

---

## 核查结果：Office 文档库（F7、F8，2026-11-26）

详细报告见根目录的 `office libs.md`。数据取自 crates.io 和 NuGet 的国内镜像，版本号和许可证是一手核实的；功能边界和保真度多数没有验证。

1. **Rust 的库按格式差别很大。** xlsx 够用：读用 `calamine`，生成用 `rust_xlsxwriter`，改既有文件用 `umya-spreadsheet`。docx 能读写，但做模板填充的库要么停更、要么只有一个版本。pptx 最弱，可用的库全在 0.x，多数是 2025-11 之后才出现的。
2. **「改既有文件」和「模板填充」在 Rust 里缺经过时间检验的库。** 无论走哪条路线，保真度验证都要算进成本。
3. **艺术字**：扫描过的 Rust 的 docx、pptx 库都没有支持的证据。
4. **C# 的免费库**：Open XML SDK 和 ClosedXML 很成熟，但偏底层；转 PDF 只有商业套件自带。**商用要避开或付费的**：EPPlus、NPOI 的官方二进制、QuestPDF、PDFsharp、iText、Xceed 的 DocX。
5. **Rust 调 C#**：最稳妥的是独立 HTTP 服务。D64 已经定了文档服务是独立服务，所以这件事对引擎不再是问题。
6. **Office 转 PDF**（D66 后置）：纯 Rust 只有一个许可证未声明的库；C# 的免费库都不渲染；常见做法是 LibreOffice，要独立进程、中文字体和按实例隔离的配置目录。
7. **Python**（主会话补查）：python-docx 1.2.0、python-pptx 1.0.2、openpyxl 3.1.5、docxtpl 0.20.2；前三个是 MIT，本机已经装着。只核实了版本和这三个的许可证。

---

## Round 12 — 第十二轮：复审之后（2026-11-26）

> 复审的结果在下一节。需要用户定的两件事在这里问；有常规做法的几件列成默认处理。
> 作答情况：AB1、AB2 已答。

### ❓ AB1 - **一个工作流同时可以有几个「已发布」的版本；发布新版本时，旧的那个怎么办**

D53 定了只有已发布的版本能被调用，D69 定了已发布的版本可以下线。但有三件事没有定，文档里因此写得不一致：一个工作流能不能同时有两个已发布的版本；发布 v2 的时候 v1 是什么状态、还能不能带着版本号调用；下线在库表里怎么表示。

(a) **同一时刻只有一个已发布的版本。** 在画布上改完点发布，新的那个上线，旧的那个自动下线；正在跑的调用照常用它开始时的版本跑完。带着版本号调用时，只认当前已发布的那个，别的返回「版本未发布」。「撤回发布」就是让这个工作流没有已发布的版本。要回滚，就把旧版本重新发布一次。
(b) **多个版本可以同时保持已发布。** 发布新版本不影响旧的；带着版本号可以调用任何一个已发布的版本；需要另外标出「默认是哪个」；下线要指明下线哪一个。

➡️ 推荐 (a)。控制面的工具和触发器都按「这个工作流现在生效的版本」走，用不到旧版本并存；多版本并存要多管一个「谁是默认」，还会让「撤回发布」有歧义（撤的是哪一个）。回滚在 (a) 里就是把旧版本重新发布，版本内容不会变。

**用户回答：** 同意推荐

**解读：** (a) ✅ [已定] → **D74**

---

### ❓ AB2 - **一条分支在等人输入、另一条还在跑时，这次调用算什么状态**

举例：一个工作流分成并行的两路。左路要弹一张确认卡等人填，右路还在调模型写摘要。这时这次调用的状态是什么？

(a) **还有东西在跑，就算「运行中」。** 等到所有能跑的都跑完、只剩等人，才变成「等待输入」。左路在等人这件事，记在它自己的等待记录里，卡片照常弹出来。
(b) **只要有一路在等人，就算「等待输入」。** 状态只是一个笼统的标签，右路照常继续跑。

➡️ 推荐 (a)。状态要如实反映「还有没有东西在跑」，控制面显示进度、判断能不能暂停都靠它。(b) 会让一次还在花钱的调用显示成「等待输入」，而且按 D49，「等待输入」状态下暂停会被拒绝，可实际上还有分支在跑。

**用户回答：** 同意推荐，没有等待事件阻塞的可以继续跑，假如左右路要合并成一路，那么合并的一路需要等左路执行完，包括等待确认卡

**解读：** (a)，并补充了一条：没有被等待挡住的分支照常推进；两路汇合时，汇合后的那一路要等左路执行完，包括等确认卡。后一条是「节点要等所有入边到齐才调度」（D29）的直接结果，用户在这里明确确认了。两条一起写进决策 ✅ [已定] → **D75**

---

### 默认处理（随本轮展示，用户未提异议，已按生效处理）

1. 重启后的处理补全：排队中的调用（`pending`）重新入队；等待输入的保持等待，重启后先检查时限，停机期间到点的按超时分支走。
2. 节点级的收尾事件：新增「节点被取消」事件（`node_cancelled`）；等待超时由引擎触发，发一条「输入已提交」事件（`action` 是 `__timeout`，`resolved_by` 是 `engine`）。
3. `__timeout` 只由引擎触发，提交输入的接口不接受它。
4. D63 的「按租户覆盖留存」有一张表存覆盖值（`tenant_retention_policies`），空的用默认值。
5. 控制面上传的入参文件和产出文件用同一套留存。
6. Dify 的 `tool` 节点在 `provider_type` 是 `workflow` 时映射到 `sub-workflow`（D22 只定了另外三种）。
7. 指向 dsh 会话变量接口的 Capability 是部署时写进去的一组预置记录，可以改、可以删，不是代码里的特例。
8. `seq`：同一次调用的序号来自同一个内存计数器，流式增量也占号；重启后从库里的最大序号续。
9. 实例级绑定的接口路径不带版本号，作用于当前的草稿版本。
10. 暂停期间提醒时限和超时时限停表，恢复时把两个时间点顺延。

第 11 条是本轮作答之后核对 D75 时补的，随下一轮开头展示：

11. 暂停期间提交的输入照常记下，恢复之后才往下调度。

---

## Round 12 结论（2026-11-26）

### 已定决策 D74–D75

| # | 决策 | 来源 |
|---|---|---|
| D74 | **一个工作流同一时刻只有一个已发布的版本**：发布新版本时旧版本自动下线；在途的 Invocation 用它开始时的版本跑完；带着版本号调用只认当前已发布的版本，别的返回 409；「撤回发布」让工作流没有已发布的版本；下线过的版本可以重新发布，回滚就是把旧版本重新发布。版本的状态是 draft、published、unpublished 三种（ADR-0014 修订） | AB1 |
| D75 | **并行分支下的调用状态**：只要有节点在执行，Invocation 就是 running；能执行的节点都执行完、只剩等人输入时，才是 waiting_input。每一路的等待各自是一条等待记录。没被等待挡住的分支照常推进；多入边的节点要等所有入边所在的分支都完成才调度，包括其中等人输入的那一支 | AB2 |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D53 | D74 | 版本的状态从 draft / published 两种变为三种，加上 unpublished；发布新版本时旧版本自动下线 |
| D69 | D74 | 下线只对当前已发布的版本有效；发布新版本也会让旧版本下线 |
| D12 | D75 | 人工输入节点在并行分支下只停它所在的那一路，其余分支照常跑；这一点与 Dify 不同（Dify 的做法待核实，OI-012） |

### 未决

- **Y4**：待定，用户倾向 (b) Python 服务先做、以后 (a) C# 服务。
- **Z2**：到迁移的时候由用户指挥（OI-017）。

### 前沿

- Y4。
- Y4 定了之后：文档服务的接口契约，以及对选定的库做保真度验证（OI-059）。
- 事实类的见 `docs/open-issues.md` 第二节。

除 Y4 和 Z2 之外，设计问题都已有了决定。本轮改动之后做了一次核对，见下面「第二次核对」。

### 第二次核对（2026-11-26，对复审的修订和 D74、D75）

方式：另一个子代理只读核对，范围比复审窄——D74、D75 是否一致地落到了各文档；复审发现表的 34 条处置是否真的到位；只在改过的部分重新找问题。

结果：

- D74、D75 都落到了各文档，没有留下旧说法。
- 34 条处置全部在声称的位置：28 条完全成立，5 条成立但有遗留，1 条部分不成立（新增的「节点被取消」事件把等待超时也写成了取消）。
- 新发现 22 条（高 1、中 10、低 11）和 7 条要确认的细节。逐条的处置在 `docs/open-issues.md` 的「第二次核对的发现」一节（OI-097 至 OI-124）。

那一条高危是 D75 的连锁影响：一路在等人、另一路在跑时，调用状态是运行中，但等待记录是真的。按调用状态去找「等人输入的调用」会漏掉它，找回待答卡和 D72 的判断都会出错。已改成一律按等待记录判断。

和上一次复审相比，这次没有「定了却没写进文档」的大类，多数是修订本身带来的连锁影响和边角情况。这一轮的修正由主会话自己核对，没有再派子代理做第三遍；实现开始之前建议再整体核一遍。

由此补的默认处理（随本轮汇报展示，用户未提异议即生效；文档已经先按它改了）：

1. **「在等人」按等待记录判断，不按调用状态。** 列表接口可以按「有未完成的等待记录」筛；找回待答卡、「一个会话只有一张确认卡」都按这个判断；提交输入时只看那个节点是不是在等待。
2. **重启后检查所有还在等待的记录的时限**，不管它所属的调用当时是什么状态。
3. **人工输入节点在并行分支下只停它所在的那一路**，其余分支照常跑。这一点和 Dify 不同（Dify 的做法待核实）。
4. **长轮询**在到终态、进入暂停、或者出现新的等待记录时返回，卡片能及时出来。
5. **执行时间按调用状态计**：一路在等人、另一路在跑时，调用是运行中，照常计时。
6. **汇合处被跳过的分支算完成**，汇合后的节点照常调度。
7. **回滚沿用当时冻结的绑定**：重新发布旧版本时仍然校验这些绑定，校验不过就拒绝发布，当前版本不变。
8. **一个工作流同一时刻只有一个草稿版本**：已经有草稿时，再新建草稿或再导入返回 409；没有已发布的版本时，新建草稿必须指定来源；没有草稿时改实例级绑定返回 409。
9. **幂等键先于版本校验**：命中幂等键就返回同一个调用，不再校验版本。

Round 12 默认处理的第 11 条也随这次汇报展示。

### 第 12 轮之后补充作答（2026-11-27）

> 仓库换回没有 dsh 源码的机器之后继续。上一台机器的 HANDOFF 留了两件要向用户展示的事，这里展示并作答。

**展示的默认处理**（上面第 11 条，加上「第二次核对」末尾的 1 到 9 条）：

1. 暂停期间提交的输入照常记下，恢复之后才往下调度。
2. 「在等人」按等待记录判断，不按调用状态。列表接口可以按「有未完成的等待记录」筛；找回待答卡、「一个会话只有一张确认卡」都按这个判断；提交输入时只看那个节点是不是在等待。
3. 重启后检查所有还在等待的记录的时限，不管它所属的调用当时是什么状态。
4. 人工输入节点在并行分支下只停它所在的那一路，其余分支照常跑。这一点和 Dify 不同（Dify 的做法待核实）。
5. 长轮询在到终态、进入暂停、或者出现新的等待记录时返回，卡片能及时出来。
6. 执行时间按调用状态计：一路在等人、另一路在跑时，调用是运行中，照常计时。
7. 汇合处被跳过的分支算完成，汇合后的节点照常调度。
8. 回滚沿用当时冻结的绑定：重新发布旧版本时仍然校验这些绑定，校验不过就拒绝发布，当前版本不变。
9. 一个工作流同一时刻只有一个草稿版本：已经有草稿时，再新建草稿或再导入返回 409；没有已发布的版本时，新建草稿必须指定来源；没有草稿时改实例级绑定返回 409。
10. 幂等键先于版本校验：命中幂等键就返回同一个调用，不再校验版本。

**同时问的一个事实：** Dify 在并行分支里的等人输入是整条工作流停住，还是只停那一路。

**用户回答：** 并行分支只停那一路

**解读：** 用户确认的是我们自己的做法：并行分支里的人工输入只停它所在的那一路，其余分支照常跑。这和 D75 以及上面第 4 条一致。Dify 自己是哪一种做法仍然没有核实，留在 OI-012，只影响 M4 的一致性夹具要不要把这种情形排除。上面 10 条默认处理，用户没有提异议，**已按生效处理**。

---

## 复审结果（2026-11-26，对 D1–D73）

方式：由一个子代理通读 grill-record，建立决策索引（含各轮的「对旧决策的修订」、各轮的默认处理、「我方做主的细节」），再逐条核对四份交付文档、ADR 正文、GLOSSARY、`docs/open-issues.md`、`HANDOFF.md`。只读。行号证据在子代理的报告里，这里只留结论；逐条的处置在 `docs/open-issues.md` 的「复审发现」一节（OI-064 至 OI-096）。

统计：29 条发现，高 2、中 10、低 17；另有 4 条需要确认的细节。按类别：拍了板没落地 9 条，文档互相矛盾 3 条，过时没标注 7 条，内部不自洽或引用指错 9 条，静默假设 1 条。

两条高危：

1. **下线的表示法，接口文档和库表不一致。** 接口按「某一个版本下线」写，还给「调用了已经下线的版本」留了错误码；库表写的是「清掉 `current_version_id`，版本状态不变」。照库表做，带着版本号仍然能调用已经下线的版本。根子是「一个工作流同时能有几个已发布的版本」从来没有定过 → OI-064，Round 12 的 AB1。
2. **没有给已有工作流新建一个版本的端点。** D59 定了引擎提供「新建版本」的 API，交付文档里只有新建工作流和更新草稿，发布 v1 之后得不到 v2。→ 已补上（OI-065）。

「拍了板没落地」的几条都是早先各轮的默认处理：D19 的节点错误处理、D55 的执行时间上限、子 Invocation 的截止时间、Round 7 的三条（`sys.dialogue_count`、新增会话变量补默认值、节点执行总数上限）、Round 5 的两条（引擎不存日志）、Round 10 默认处理第 7 条（运行时计算的鉴权）、`user_id` 没出现在提交请求里。都已补进交付文档。

过时的说法：「画布读写的是同一份定义」（D67 之后不成立，ADR-0010 已加修订）；四份交付文档的标题还写 D1–D63；ADR-0009 的例子还是「http-request → http-tool」。都已改。

4 条需要确认的细节：并行分支下的调用状态（要用户定 → OI-093，AB2）；「内置的 Capability」指什么、上传的入参文件留多久、`seq` 怎么分配（按默认处理，见 Round 12）。

专项核对里成立的：D36、D37、D38、D45、D46、D48、D56、D58、D64、D65、D67、D71、D72、D73 都有落点；D53、D54 除了上面两条都能对上；OI 编号连续。

教训和 Round 5 一样：宣布设计树走完之前必须做复审，这次也还没有走完。

---

## 交付文档修订（2026-11-26，用户就寝前）

用户睡前要求把能做完的做完并提交。四份交付文档按 D1–D63 重写了一遍，原来的「已知过时点」清单不再需要：

| 文档 | 主要变化 |
|---|---|
| `docs/db-schema.md` | `dsl_yaml` 换成 `definition`；版本有 draft / published；Node Run 身份；在途节点的状态；会话绑定工作流、对话轮次表；Provider 配置表；文件表；留存策略；撤回多实例的说法 |
| `docs/api-spec.md` | 暂停与恢复的语义；`timeout_ms` 只算运行时间；长轮询；定义与发布的 API；Node Binding 的 API；Provider 配置与密钥的 API；事件信封带 `node_run_id`、回放只含结构性事件；导入三级严重度 |
| `docs/dify-node-mapping.md` | 版本区间与 `.ifpkg`；`end` 的更正；模型映射与记忆；知识检索默认是阻断项；环境变量；应用类型的处理；完整的导入流程 |
| `docs/milestones.md` | Node Binding 路由提前；新增「定义与发布」「文件」两条线；补 dsh 接入线；准入控制与留存 |

**修订稿里我方做主的实现级细节**（没有单独出题，不同意再改）：

1. 事件新增三个：`waiting_input_reminder`（提醒时限到）、`invocation_paused`、`invocation_resumed`。
2. 类型级绑定的 API 改为 `PUT /node-bindings/defaults`。初稿把它挂在单个工作流下（`PUT /workflows/{id}/bindings`），和「类型级」的语义对不上。
3. 删掉 `capability_bindings` 表：节点对 Capability 的引用已经在定义里。
4. 对话轮次单独一张表 `conversation_turns`。
5. 节点执行记录的状态里有 `unknown`，对应 D36 预留的「挂起待裁决」。
6. Dify 0.7.0 新增的 `agent` 应用模式按错误级处理，文档里标了「待确认」。
7. crate 划分里新增 engine-publish、engine-egress、engine-conversation、engine-files，并注明是实现期建议。
8. 节点映射表加了一节「一致性验证」，作为实现期建议。

**文档里标「待 Round 9」的部分**：文件怎样在引擎与外部服务之间传、文档节点的形态与实现、dsh 接入线的验收。

---

## 交付文档修订（第 9、10 轮作答之后，2026-11-26）

按 D64–D70 补上了四份交付文档里原先标「待 Round 9」的部分：

| 文档 | 主要变化 |
|---|---|
| `docs/api-spec.md` | 提交调用的请求增加 `session_ref`，列表可按它筛；新增下线的端点；`tenant_id` 来自环境变量；默认的执行时间上限 1 小时；第 5 节写清文件在一次 HTTP 调用里怎么进出、dsh 一侧由控制面搬运；错误码增加「版本已下线」 |
| `docs/db-schema.md` | `invocations` 增加 `session_ref`；`current_version_id` 为空的含义加上「已经下线」；Capability 的 `spec` 里写文件参数的传递方式；文件一节指向 api-spec；token 一节注明引擎单独记账 |
| `docs/dify-node-mapping.md` | 文档节点一行改为「经 Capability + http-tool 接入，本期支持三类操作」；注明画布图要经转换器、`userinput` 落到 human-input、知识检索后端的响应适配放在哪还没定 |
| `docs/milestones.md` | M2 加下线；M5 加企业内网的出口层要求；M6 写清文档服务的形态和验收；M8 按转换器、迁移、工具注册、文件搬运、待答卡、外部网关拆开，验收给了草案；风险表加两行 |

其它落盘：`docs/open-issues.md`（按用户要求，把查出来的问题全部记入，OI-002 至 OI-058）；`GLOSSARY.md`（下线、dsh 会话变量）；`docs/adr/0015`（新增）；ADR-0005、ADR-0014（修订）。

这次我方做主、没有单独出题的：

1. M8 的验收是草案，等 OI-002、OI-017 定了再确认。
2. 「dsh 会话变量」这个叫法是为了和 Conversation 的会话变量区分，是我提的。
3. Round 11 起题目编号改用两个字母，从 AA 开始。

---

## 交付文档修订（第 11 轮作答之后，2026-11-26）

按 D71、D72 和 Round 11 的两条默认处理更新：

| 文档 | 主要变化 |
|---|---|
| `docs/api-spec.md` | 新增 `GET /invocations/{id}/node-runs`（取节点执行记录，供控制面做事后排查和「对已完成的运行的报告做修改」这类工具）；列表 API 加排序；Capability 一节写明响应适配放在后端前面的适配服务里（D71）；契约要点加一条「一个 dsh 会话里同时只有一个等人输入的调用，是控制面自己的限制」（D72） |
| `docs/dify-node-mapping.md` | 知识检索的说明改为「响应适配放在各后端前面的适配服务里」 |
| `docs/milestones.md` | M5 加各后端前面的适配服务；M8 加「Python 第三方工具包成 HTTP 服务」和「一个会话只有一张待答卡」，从没定的清单里去掉多张卡那一条 |
| `docs/adr/0005` | 加一条修订：Capability 可以指向后端前面的适配服务 |

其它落盘：`office libs.md`（新增，F7、F8 的详细报告和 Python 补查）；`docs/open-issues.md`（关掉 OI-005、OI-006、OI-007、OI-008、OI-013，新增 OI-059 到 OI-062）。`docs/db-schema.md` 没有受这一轮影响。

Z3 作答之后（D73）又同步了一次：`docs/api-spec.md` 写明引擎里有一组指向 dsh 会话变量接口的内置 Capability、`session_ref` 要能被节点引用；`docs/db-schema.md` 注明 dsh 会话变量不存引擎库；`docs/dify-node-mapping.md` 加一行说明 `vars` 节点的落点；`docs/milestones.md` 的 M8 加「dsh 侧要开带服务令牌的接口」；`GLOSSARY.md` 的「dsh 会话变量」补上「工作流经 dsh 的接口读写」；`docs/open-issues.md` 关掉 OI-002，新增 OI-063。

---

## 交付文档修订（复审之后，2026-11-26）

按复审结果改的，逐条对应 `docs/open-issues.md` 的「复审发现」一节：

| 文档 | 主要变化 |
|---|---|
| `docs/api-spec.md` | 标题和依据更新到 D1–D73；提交请求补上必填的 `user_id`，`tenant_id` 改为由引擎读环境变量；新增给已有工作流新建草稿版本的端点，导入可以挂在已有工作流下；实例级绑定的路径去掉版本段；`/inputs` 不接受 `__timeout`；写明到执行时间上限后的处置和子调用的截止时间；事件表加 `node_cancelled`，`input_submitted` 写明超时时的形态；契约要点加序号的分配、运行时计算的鉴权、引擎不存日志 |
| `docs/db-schema.md` | 标题和依据更新；可空的 `tenant_id` 进唯一键时包一层 `IFNULL`；`session_ref` 和 `timeout_at` 的注释跟上 D73；补上重启后 `pending`、`waiting_input` 两态的处理、子调用的截止时间、`sys.dialogue_count`、新增会话变量补默认值、节点执行总数上限；新增 `tenant_retention_policies` 表，上传的入参文件并入留存；给和接口文档不一致的那条「下线」说明加了警示，等 OI-064 |
| `docs/dify-node-mapping.md` | 标题和版本策略更新；表头说明「可绑定」都指实例级；`is_parallel` 更正为 `parallel_nums`；`tool` 的 `workflow` 一类注明是默认处理；第 6 节的标题改为「应用类型」；新增第 9 节「节点错误处理」（D19），导入流程加一步；「画布读写同一份定义」改掉 |
| `docs/milestones.md` | 标题更新；M1 加执行时间上限、节点执行总数上限、按租户覆盖的留存和对应的验收；M2 加新建草稿版本、静态引用校验；M3 加共享配置的快照，验收注明依赖 M4；M4 加节点错误处理；M5 加运行时计算的鉴权；M6 里 code 节点和子工作流的入口挪到 M7；M7 加文件回传和子调用的截止时间；M8 加 LLM 直连、工具超时上限、会话工具的两个接口；M9 加日志；风险表更新 |
| `GLOSSARY.md` | Workflow Definition 不再说是「画布编辑」的依据；Dify DSL 加上 `.ifpkg`；Invocation 和 Tenant 各补一句；dsh 会话变量补上「工作流经 dsh 的接口读写」 |
| ADR-0001、0008、0009、0010、0015 | 各加一条修订或更正：鉴权没有「后续 ADR」；占位符「只在出口这一步替换」，运行时计算的鉴权不进占位符机制；绑定的例子不再写「http-request → http-tool」；「画布读写的是同一份定义」作废；三级严重度归 D53 |

这次我方做主、没有单独出题的，就是 Round 12 的默认处理：第 1 到 9 条来自复审的发现和待确认项（第 9 条对应 OI-079），第 10、11 条是落 D75 时补的。第 1 到 10 条已随 Round 12 的汇报展示，第 11 条随第二次核对的汇报展示。

AB1、AB2 作答之后（D74、D75）又同步了一次：

| 文档 | 主要变化 |
|---|---|
| `docs/api-spec.md` | 提交调用时带版本号的规则和错误码；发布新版本时旧版本自动下线；下线只对当前已发布的版本有效；长轮询里「等待输入」的含义；暂停期间提交的输入怎么处理；契约要点加版本规则和调用状态的判定 |
| `docs/db-schema.md` | 版本的状态扩成三种；`current_version_id`、`workflow_version_id` 的注释；重写了发布、下线、回滚的规则，删掉了那条带警示的旧说法；加了调用状态怎么取、汇合处怎么等、暂停时时限怎么顺延 |
| `docs/milestones.md` | M1 加并行分支下的调用状态和对应的验收；M2 的验收加发布 v2 后 v1 自动下线、带着 v1 调用被拒、回滚 |
| `GLOSSARY.md` | Workflow Version 的状态改为三种；Unpublish 的定义补上「发布新版本时旧的那个自动下线」和「回滚就是这么做的」 |
| `docs/adr/0014` | 加一条修订：同一时刻只有一个已发布的版本 |
| `docs/open-issues.md` | 关掉 OI-064、OI-093、OI-023 |

---

## Round 13 — 第十三轮：文档服务的接口契约（2026-11-27）

> 用户在换回没有 dsh 源码的机器之后要求问这一轮。依据：D64（文档节点 = Capability + Http-Tool Node）、D65（文件内联或预签名地址）、D66（本期支持模板填充、从零生成、编辑既有文件）、`office libs.md`、`dsh workflow.md` 第 9 节。
> Y4（实现语言）仍待定，用户倾向 Python 服务先做、以后换 C#。本轮的题目都不依赖语言：接口固定下来，换实现只改 Capability 指向的服务。
> 出题前先说了两个名字：**文档服务**是一个独立的小服务，专门生成和修改 Word、Excel、PPT，工作流通过一个 http-tool 节点调用它；**文档接口**是它对外提供的那几个 HTTP 接口。
> 作答之后用户在第 12 轮之后展示的默认处理上做了两条批注，一并记在本轮末尾。

### ❓ AC1 - **文档服务对外提供几个接口**

Capability 是一个接口登记一条，http-tool 节点在画布上是从下拉框选一个，选完看到它的入参表单。

(a) **一个接口，请求里写明做哪种操作**：`POST /documents`，请求里有 `operation` 字段
(b) **一种操作一个接口**：填充、从零生成、编辑既有文件各一个
(c) **按文件格式切**：`/docx`、`/xlsx`、`/pptx`，请求里再写要做什么

➡️ 推荐 (b)。画布上会有三个清楚的选项，每个的入参表单不一样：填充要选模板，生成要给数据结构，编辑要给原文件。(a) 是一个表单挤着三种操作的参数，画布做不出联动。(c) 会把同一种操作的参数抄三遍。以后加第四种操作，(b) 只是多登记一条。

**用户回答：** B ✅ [已定] → **D76**

---

### ❓ AC2 - **模板从哪来**

填充要告诉服务「用哪个模板」。比如一份合同模板，多个工作流都要用。

(a) **每次调用都把模板文件传过去**：模板是工作流里的一个文件变量
(b) **模板存在文档服务里，调用时只报模板的名字**：管理员先把模板传给文档服务，节点上从列表里选
(c) **模板存在引擎的文件存储里，调用时传模板的地址**：文档服务用引擎给的地址去取

➡️ 推荐 (c)。引擎已经有文件存储，模板当成一种带名字的文件存进去，画布上按名字选。这样模板能跨工作流复用，改一次全部生效，文档服务也不用自己做一套模板管理。

**用户回答：** 同意推荐 ✅ [已定] → **D77**

---

### ❓ AC3 - **数据怎么告诉服务**

比如合同要填甲方、乙方、金额。

(a) **一份普通的 JSON**：键是模板里占位符的名字，值是要填的内容。节点上把每个占位符映射到一个上游节点的输出，引擎解析好再发
(b) **把工作流的变量引用原样传过去**：让文档服务自己去找变量的值

➡️ 推荐 (a)。(b) 不行：文档服务在引擎之外，看不懂工作流的变量引用。配套一条：文档服务提供一个「这份模板有哪些占位符」的查询，画布据此生成填表单，不用人手敲占位符的名字。

**用户回答：** 同意推荐 ✅ [已定] → **D78**

---

### ❓ AC4 - **编辑既有文件：改什么怎么描述**

给它一份已有的 PPT，要它「末尾追加一页」。

(a) **一组固定种类的修改动作**：比如替换文字、追加段落、追加一页、改单元格。请求里给一个动作列表，服务按顺序执行
(b) **让模型来决定怎么改**：把原文件和「把标题改成 X」这样的自然语言要求一起交给服务
(c) **请求里带一段代码**，服务执行它来改文件

➡️ 推荐 (a)。(b) 的结果不稳定，token 花在哪也不清楚。真要让模型决定改法，在工作流里放一个 LLM 节点，让它输出 (a) 那样的动作列表。(c) 等于在文档服务里再开一个代码沙箱，引擎自己已经有代码节点了。

**用户回答：** 同意推荐 ✅ [已定] → **D79**

---

### ❓ AC5 - **做错了怎么办**

比如模板里有个占位符，数据里没给；或者要改的位置不存在。

(a) **一律拒绝，整个请求失败**：返回错误和出错的地点，不产出文件
(b) **能容错的就容错**：缺的留空、找不到的跳过，把文件生成出来，另附一份「哪些没做成」的清单

➡️ 推荐 (a)。一份「金额那栏是空的」合同流出去，比节点失败糟糕得多。要留空，就在工作流里显式填一个空字符串。

**用户回答：** 同意推荐, 如果用户再次同意，那么传入特殊占位字符串代表空值

**追问（我方）：** 我的理解是：默认遇到缺值就失败；如果用户在节点上明确打开一个开关表示「允许留空」，那么缺的值填一个约定的特殊字符串，不再报错。这个理解对吗？那个「特殊占位字符串」具体是什么，空字符串还是一个看得出来的标记？

**用户回答：** 对；看得出来的标记，你来设计

**解读：** 默认失败 ✅。节点上可以显式打开「允许留空」；打开后缺的值填一个**看得出来的标记**，由我方设计，见下面的设计 → **D80**。用户原话里的「如果用户再次同意」我按「在工作流里明确表示同意（打开开关）」理解。

**我方设计的标记（用户授权设计，可被推翻）：**

- 缺的值一律填 `{{EMPTY}}`。选它是因为它在 Word、Excel、PPT 里都是合法的普通文本，肉眼看得出是「这里本来该有值」，而且和模板填充的占位符长得像，一眼能认出是没填上。
- 这个标记由**文档服务**在「允许留空」打开时写入，工作流这边不参与：节点把开关传给服务，服务遇到缺值就填它。
- 只对「模板里有占位符、数据里没给」生效。其它错误照旧失败：要改的位置不存在、给的文件格式不对、模板本身打不开。
- 标记的写法固定为 `{{EMPTY}}`，文档服务的每个实现（Python、C#）都必须一致，这样换实现不影响已产出的文档的样子。
- 要真的留空，就在节点上把那个占位符显式映射成一个空字符串，不打开「允许留空」。

---

### ❓ AC6 - **做好的文件怎么交回来**

(a) **沿用已定的文件传递规则**：小的内联在响应里，大的用引擎给的预签名地址上传。工作流拿到的是一个文件引用
(b) **文档服务自己存文件**，返回它那边的地址，引擎再去下载

➡️ 推荐 (a)。这是 D65 的直接结果，产出文件一律经引擎的文件存储。

**用户回答：** A ✅ [已定] → **D81**

---

### ❓ AC7 - **现有的 Markdown 转 docx 怎么并进来**

dsh 现在的文档能力只有这一种（学术模板）。Y3 的作答里用户把它归到了模板填充这一类（D66）。

(a) **作为文档服务的一个独立接口**：入参是 Markdown 文本和模板名，出参是 docx
(b) **不并进来**，维持现状，它自己单独包成一个 HTTP 服务
(c) **硬拆成模板填充**：先把 Markdown 塞进模板的正文占位符

➡️ 推荐 (a)。(c) 做不到：模板填充只会把值填进占位符，填进去的 Markdown 还是纯文本，不会变成标题和表格。「把 Markdown 排版成 Word」必须是服务自己的一项能力。(a) 和「属于模板填充那一类」不冲突：对用户来说它就是套模板出文档，只是接口上要单独一个。现在的 Python 实现可以直接作为文档服务的第一个接口。

**用户回答：** A ✅ [已定] → **D82**

---

### ❓ AC8 - **保真度验证**

模板填充和编辑之后，没被碰到的元素会不会丢，比如字体、表格样式、页眉页脚、修订痕迹、文本框。`office libs.md` 写明了这些库都没有验证过。

(a) **现在就定「验证什么」**：写一份和语言无关的清单，拿哪几种真实模板、检查哪些元素、什么算通过。等实现语言定了再照着做
(b) **等实现语言定了再说**

➡️ 推荐 (a)。清单定了还有个好处：选语言的时候，Python 和 C# 的库拿同一份清单去试，比凭印象可靠。需要用户提供两三种真实在用的模板。

**用户回答：** A；实现语言待定，模板别的机器有

**解读：** (a) ✅ [已定] → **D83**。Y4 继续待定（OI-003 不变）。真实的模板在另一台机器上，清单先按通用情形写，到了那台机器再对着真实模板校准 → 新增 **OI-125**。

---

## Round 13 结论（2026-11-27）

### 已定决策 D76–D83

| # | 决策 | 来源 |
|---|---|---|
| D76 | **文档服务一种操作一个接口**：模板填充、从零生成、编辑既有文件各是一个 HTTP 接口，各登记成一条 Capability。以后加操作只多登记一条，不改已有接口 | AC1 |
| D77 | **模板存在引擎的文件存储里**：模板是一类带名字的文件，画布上按名字选，能跨工作流复用；调用时引擎把模板的地址传给文档服务，文档服务不自己做模板管理 | AC2 |
| D78 | **数据是一份普通 JSON**：键是占位符的名字，值由节点映射到上游节点的输出，引擎解析后再发；文档服务另提供一个「这份模板有哪些占位符」的查询，画布据此生成填表单 | AC3 |
| D79 | **编辑既有文件用一组固定种类的修改动作**：请求里给动作列表，服务按顺序执行；不让模型在服务内部决定改法，也不在服务里执行代码 | AC4 |
| D80 | **出错默认整个请求失败**，返回错误和出错的地点，不产出文件。节点上可以显式打开「允许留空」：打开后，模板里有而数据里没给的值填 `{{EMPTY}}`；其它错误照旧失败 | AC5 |
| D81 | **做好的文件沿用已定的传递规则**：小的内联，大的用预签名地址上传，工作流拿到的是一个 File 引用；文档服务不自己存文件 | AC6 |
| D82 | **Markdown 转 docx 作为文档服务的一个独立接口**：入参是 Markdown 文本和模板名，出参是 docx。对用户来说属于模板填充那一类，但接口上单独一个，因为「把 Markdown 排版成 Word」不是占位符填充做得到的 | AC7 |
| D83 | **现在就定保真度验证的清单**，和语言无关；实现语言定了再照着做，选语言时各候选的库拿同一份清单去试 | AC8 |

### 对旧决策的修订（原文保留未改）

| 旧决策 | 被谁修订 | 改了什么 |
|---|---|---|
| D66 | D82 | 「现有的 Markdown 转 docx 属于模板填充」保留；接口上它是独立的一个，不走模板填充接口 |
| Round 11 默认处理第 1 条 | D82 | 文档生成工具不再「各自包成 HTTP 服务」，而是并进文档服务；两个检索工具那条不变 |

### 仍待定

- **Y4 实现语言**：用户明确继续待定（OI-003）。
- **Z2 存量迁移**：到迁移的时候由用户指挥（OI-017）。

### 对默认处理的两条批注（2026-11-27）

用户看了第 12 轮之后展示的 10 条默认处理，做了两条批注。

**批注一**，针对第 2 条里的「一个会话只有一张确认卡」：

> 「可以有多个」
> 「D72 定的是『一个会话里同时只有一个等人输入的调用』——这个是对的，一个工作流里可以有多个确认卡，但是同时显示给用户看只有一个」
> （回答我方的问题「多张确认卡，改到什么程度」）「dsh 对话框与用户交互时只有一个，工作流可以有多个确认卡」

**解读：** D72 **不变**——一个会话同时只有一个等人输入的**调用**，这条用户明确说是对的。变的是卡片这一层，定下三件事：

1. 一个工作流里可以同时有多张确认卡（并行分支里两路各在等人，就是两张）。这一点 D75 已经蕴含，用户在这里明确确认。
2. 同时显示给用户看的只有一张。dsh 的对话框同一时刻只和用户做一次交互。
3. 多张卡时先显示哪一张、显示中的卡提交之后下一张什么时候出来，用户没有指定，由我方设计，见下。

→ **D84**（不修订 D72，是 D72 的细化）

**我方设计的显示规则（用户授权设计，可被推翻）：**

- 顺序：按等待记录产生的时间，先产生的先显示。
- 前一张卡被提交（或者走到超时分支）之后，才显示下一张。不并排显示。
- 显示中的那张卡只挡它自己那一路。这个工作流里其它在跑的分支照常跑（D75）；会话里别的工作流不受影响。
- 其余的卡在等的时候照常计各自的提醒时限和超时时限，不因为「还没轮到显示」而暂停计时。
- 卡片的字段没有全部渲染这笔欠账（OI-041）不再因为「只有一张卡」而保留，要还。

**批注二**，针对第 7 条「汇合处被跳过的分支算完成，汇合后的节点照常调度」：

> 「没看懂」

我方用一个图重讲了：一个判断节点分出两条路，只走其中一条；另一条路不会执行。汇合点要等所有入边到齐才能开始，如果死等那条永远不会跑的路，汇合点就永远开始不了。所以那条「根本不会跑」的路对汇合点算「到齐了」。和 AB2 的区别：一条路**真的要跑**（比如在等确认卡）→ 汇合点等它；一条路**根本不会跑** → 汇合点不等。

**用户回答：** 同意

**解读：** 规则不变，用户确认 ✅。按约定把这句话在各文档里改成带图的讲法。

### 派生的待办

- 前端：一个工作流里有多张待答卡时按上面的规则逐张显示；卡片渲染全部字段（OI-041 要还）→ 记入 OI-022。
- 「允许留空」的开关是节点配置的一部分，要进 http-tool 节点引用的 Capability 的入参定义 → 实现期处理。
- 真实的模板在另一台机器上，保真度验证的清单到了那台机器上再对着它们写实 → OI-125。
- 落库表文档时发现一个没人定过的设计选择，我方一度顺手写成了一条规则，随即撤回，改记成待问的问题：**模板改了之后，正在用它的已发布工作流是立刻用新内容，还是要重新发布才生效** → OI-126，还没有问过用户。

### 本轮落盘时我方做主的细节（没有单独出题，用户可推翻）

1. `files` 表加了 `kind` 一列（uploaded / produced / template），模板不随调用过期，同一租户下按名字唯一。D77 只定了「模板是引擎文件存储里一类带名字的文件」，列怎么设计是我定的。
2. 多张卡的显示规则（见「批注一」下面「我方设计的显示规则」）。用户授权了设计，但没有逐条确认。
3. `{{EMPTY}}` 这个标记的具体写法（D80）。用户授权了设计（「你来设计」），但没有确认就用了这个字符串。
4. OI-126 是我方在落文档时发现的，不是用户提出来的。

### Round 14 前沿

设计问题除下面三件之外都有了决定：

1. **OI-126** 模板改了之后什么时候生效（没问过用户）。
2. **Y4** 实现语言（OI-003，用户两次说待定）。
3. **Z2** 存量迁移（OI-017，到迁移的时候由用户指挥）。

Y4 定了之后：文档服务的接口契约细化（每个接口的请求与响应字段）和保真度验证（OI-059，依赖 OI-125 的真实模板）。
