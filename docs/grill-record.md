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

> 本轮进行中：X1 已答，X2–X4 待答。回答到齐后补全结论。

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

### ❓ X2 - **工作流产出文件怎么做**（待答）

你说有产出文件的需求，W3 里「引擎对文件只读」的前提就不成立了。

① **文件存储**：引擎加一个可插拔的文件存储接口（对象存储，本地开发用本地目录），和出口层一样按部署档位配置。节点产出的文件一律经它写入，按租户和 Invocation 分前缀
② 引用沿用 W3 定的形态。引用里的地址是存储的内部地址，对外访问由控制面换签名链接
③ 本期哪些节点能产出文件：
(a) HTTP 节点拿到二进制响应
(b) code 节点写出文件，worker 把文件交回引擎
(c) 子工作流把文件回传给父级
(d) 模型产出，比如生图、生视频

➡️ 推荐答案：①② 如上；③ 本期做 (a)(b)(c)，(d) 取决于厂商接口会不会返回文件，后置。

---

### ❓ X3 - **Office 文档的生成和编辑放哪一层**（待答，依赖 F7）

(a) **code 节点**：Python worker 预装上面那些成熟库，引擎不为 Office 写专用节点
(b) **Rust 内置节点**：只做库撑得住的，比如 xlsx 的读写、docx 和 pptx 的模板填充
(c) **Capability**：外部的文档服务。Office 转 PDF 据我所知在 Rust 里得靠 LibreOffice 这类外部引擎，复杂编辑也一样

➡️ 推荐答案：先 (a)+(c)。(b) 等 (a) 里出现重复的模式再沉淀成节点。子代理核实后如果 Rust 库比我记的成熟，再调整。

另请用户举例：产出 Office 文档的典型场景（模板填充、报表导出、格式转换、改既有文件），用来判断 (b) 值不值得做。

---

### ❓ X4 - **运行数据留存多久**（待答）

每次 Invocation 会留下调用记录、节点执行记录、事件、变量快照、轮次、token 明细，现在还有产出的文件。全部永久保留的话，库会无限长大。

(a) 不清理
(b) **分类留存**：变量快照在 Invocation 进入终态后删除，续跑不再需要它；调用记录、节点执行记录、事件、轮次按可配的保留期清理，默认 180 天；token 明细留 2 年；产出文件默认留 7 天；都支持按租户覆盖
(c) 在 (b) 之上加一个审计档：租户可以开启「保留变量快照直到调用记录过期」

➡️ 推荐答案：(b)。D7 选每节点快照时把「审计」列为了理由，但事后排查需要的是每个节点的输入输出，那在节点执行记录里，照常保留；变量快照只是续跑用的。如果确实需要事后看到每个节点的完整变量上下文，就选 (c)。

---

### 默认处理（随本轮展示，待用户无异议后生效）

W1 只答了「两个数字」是哪两个，下面几条是默认处理：

- 工作流执行时间上限只累计运行中的时间：从开始运行起算，等用户输入和暂停期间不计时，它们各有自己的时限；排队等起跑的时间也不算
- 到上限时 Invocation 判失败（错误码 TIMEOUT），在途节点和子工作流一起取消
- 配置优先级是引擎默认值、工作流定义里的值、提交调用时的 `timeout_ms`，后者覆盖前者
- human-input 节点上的提醒时限（V1 定的）保留为可选项，工作流级别不设提醒时限
- 队列先进先出，不做优先级

---

## Round 8 结论（进行中）

### 已定决策 D60

| # | 决策 | 来源 |
|---|---|---|
| D60 | **Dify 版本策略**：接受 `0.1.5` 至 `0.7.0`，靠内容判断文件在插件化分界之前还是之后；比 0.7.0 新的文件照常导入并在报告置顶警告，不拒绝；导入接口一并接受 `.ifpkg`（解包取 DSL，资源忽略）。无论版本号是多少，都按节点 `data.type` 对照引擎支持的类型校验。下限不是硬需求，实现期可抬到 0.2.0 | X1 |
