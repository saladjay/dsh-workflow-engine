# D. 实施里程碑（修订稿，对应 D1–D84）

> 设计输入：grill 决策 D1–D84（见 `docs/grill-record.md`）、ADR-0001~0015。
> 和 Round 4 的初稿相比：Node Binding 的路由提前到地基里；新增「定义与发布」和「文件」两条线；补了 dsh 接入线；发布门槛和绑定冻结进了 M2。
> 还没定的：M6 里文档服务的实现语言（Y4）；M8 的验收（等 OI-017）。

## 1. Cargo workspace 结构

```
engine/
├── crates/
│   ├── engine-model          # Workflow Definition（Dify DSL 的结构 + 扩展命名空间）的类型、
│   │                         # 节点契约、变量引用、事件枚举
│   ├── engine-core           # 事件驱动 DAG 调度、Node Run 身份、Invocation 状态机、变量上下文、
│   │                         # Node Binding 路由、sub-workflow、准入控制、队列抽象、重启恢复
│   ├── engine-dify-import    # Dify 导入：解包、版本区间与归一化、节点类型校验、三级报告、模型映射
│   ├── engine-publish        # 草稿与发布：阻断项检查、绑定解析与契约校验、生成内部图表示
│   ├── engine-nodes          # 内置节点：if-else / aggregator / assigner / list-operator /
│   │                         # template-transform / iteration / loop / start / end / answer / document-extractor
│   ├── engine-llm            # Provider 主干与适配器、结构化输出分层、流式、记忆（对话轮次）、token_usage
│   ├── engine-capability     # Capability Registry、http-tool 与 http-request、MCP 远程客户端
│   ├── engine-egress         # 出口层：gateway / local 两种档位、占位符、SSRF 校验、密钥库
│   ├── engine-humaninput     # 等待记录、提醒与超时、唤醒
│   ├── engine-conversation   # Conversation、会话变量、对话轮次、会话内排队
│   ├── engine-files          # 文件存储接口、File 引用、产出文件的收集、与外部服务的文件交换
│   ├── engine-code           # JS 池、Python worker 池管理
│   ├── engine-agent          # agent 节点的基础循环
│   ├── engine-store          # 仓储 trait、MySQL 实现、事件落库与回放、留存清理
│   └── engine-api            # HTTP / SSE、鉴权中间件
├── python-worker/            # Python worker 宿主（main shim + 长度前缀 JSON 协议）
├── migrations/               # sqlx 迁移（对应 docs/db-schema.md）
└── Cargo.toml                # workspace
```

crate 的划分是实现期的建议，可以合并。Office 文档服务是 workspace 之外的独立 HTTP 服务，一种操作一个接口，各登记成 Capability（D64、D76）；实现语言待定（第 9 轮的 Y4）。

**参考库使用约定**：`rig`（MIT/Apache）、`swiftide`（MIT/Apache）可以按组件借用或参考；`graphbit` 是 source-available 许可，法务确认前只作架构参考，不复制代码、不直接引依赖；`pi_agent_rust` 同为参考。

## 2. 里程碑

### M0 地基

workspace 搭建；`engine-model` 解析 Workflow Definition；`engine-core` 的事件驱动调度器、Node Run 身份、Node Binding 路由的接口；内存队列；内存仓储；最小的提交与查询 API；一个 echo 工作流端到端跑通。

**验收**：提交、调度、完成、查状态，SSE 收到 `invocation_finished`。循环和并行迭代里，每一次节点执行都有各自的 `node_run_id`。

### M1 持久化与恢复

MySQL 仓储与迁移；事件只在节点边界落库，节点开始先落库再执行；流式增量只广播；取消、暂停、恢复；并行分支的调度和调用状态的判定（D75）；最小可用的 human-input 节点（挂起、提交、继续；两级时限和 assignees 在 M5）；执行时间上限，只算运行时间（D55）；重启续跑（至少一次，幂等头随出站请求下发）；SSE 回放结构性事件；幂等键；准入控制，含每个 Invocation 的节点执行总数上限（Round 7 的默认处理）；留存清理任务，含按租户覆盖。

**验收**：`kill -9` 之后，运行中的 Invocation 从快照续跑成功，HTTP 后端凭幂等头没有重复执行；暂停中的保持暂停；排队中的调用重新入队；回放不缺结构性事件；到了执行时间上限的调用判失败，在途的节点被取消；一路在等人输入、另一路还在跑时，调用是运行中，照样可以提交输入；两路汇合的节点等两路都完成才继续。

### M2 定义与发布

定义 API（新建工作流、给已有工作流新建草稿版本、更新草稿、发布、下线）；Dify 导入（版本区间、归一化、节点类型校验、静态引用校验、三级报告、模型映射、环境变量）；Provider 配置管理；Node Binding 的发布时冻结与契约校验。

**验收**：导入 0.1.5 到 0.7.0 的几份真实 DSL，报告的分级准确；带阻断项的版本发不了布；改了绑定，已发布版本的行为不变；发布 v2 之后 v1 自动下线：带着 v1 的版本号调用被拒绝，不带版本号的调用拿到 v2，在途的调用仍按 v1 跑完，把 v1 重新发布就是回滚（D74）；下线之后新的调用被拒绝，在途的照常跑完。

### M3 LLM 线

Provider 主干配置化接入；流式；结构化输出分层；`token_usage`；llm、parameter-extractor、question-classifier 节点；记忆（对话轮次）；Conversation、会话变量、会话内排队；共享运行配置在调用开始时解析并留快照（D54）。

**验收**：三家国内厂商配置化接入；开了 memory 的节点能看到历史轮次，没开的完全不受影响（这一条依赖 M4 的 answer 节点，两个里程碑并行时要错开验收）；会话变量不丢更新。

### M4 Dify 兼容线

节点语义对齐：if-else、aggregator（含别名归一）、assigner、iteration（并行加三种错误模式）、loop（三种终止）、list-operator、template-transform、answer、trigger-* 转控制面配置、document-extractor 的内置格式；节点错误处理，即节点级的重试、fail-branch、default-value（D19）。一致性夹具。

**验收**：对照夹具，执行结果与 Dify 一致，包括配置了重试和异常处理策略的节点。

### M5 集成线

Capability Registry；http-tool 与 http-request；出口层的两种档位与占位符；网关联调；SSRF 防护，含企业内网的私网白名单、Host 头改写、自签证书的信任（OI-026）；MCP 远程客户端；human-input 全链路（两级时限、assignees、跨重启唤醒，暂停期间时限停表）；知识检索经绑定对接，各后端前面的适配服务（D71）；需要运行时计算的鉴权不进出口层，这类后端包成 HTTP 服务（Round 10 的默认处理第 7 条）。

**验收**：审批场景跨引擎重启完成；暂停期间提醒时限和超时时限停表，恢复后顺延；占位符经网关替换成功，替换失败被拒；`local` 档下没有网关也能跑通。

### M6 文件线

文件存储接口；File 引用；产出入口（HTTP 二进制响应、生图；code 节点和子工作流回传的入口在 M7 接上）；与外部服务的文件交换（小文件内联，大文件用预签名地址，D65）；LLM 视觉输入内联；Office 文档服务（独立的 HTTP 服务，经 Capability 接入，D64；五种接口——填充、生成、编辑、把 Markdown 排版成 Word、查询模板的占位符，D76、D82；模板是引擎文件存储里一类带名字的文件，D77；默认出错即失败，「允许留空」时缺值填 `{{EMPTY}}`，D80；实现语言待 Y4）；文件留存。

**验收**：工作流产出的 Word、Excel、PPT、图片能经签名链接下载；到期被清理；文档服务能对一个 docx 模板做填充、从零生成一份 xlsx、给一份已有的 pptx 追加一页、把一篇 Markdown 排成 docx，四件产物都能下载；模板里的占位符缺值时请求失败、不产出文件，打开「允许留空」后同一请求产出文件且缺值处是 `{{EMPTY}}`；换一个实现的文档服务指向同一组 Capability，工作流不用改。保真度验证按 `docs/open-issues.md` 的 OI-059 所列的清单做，等实现语言定了再执行。

### M7 沙箱与扩展

Python worker 池（python-build-standalone、uv 预装、白名单、seccomp / cgroups）；JS 池；code 节点，含把生成的文件交回引擎；agent 基础循环；sub-workflow 的深度上限、不重复派生、文件回传，以及子 Invocation 的截止时间不超过父 Invocation 的剩余时间（Round 5 的默认处理）。

**验收**：Dify 的 code 节点原样导入执行；死循环或超内存的代码被强杀，不影响引擎。

### M8 dsh 接入线

前置 F3 已查清（2026-11-26，事实见 `dsh workflow.md`）：dsh 现有的工作流是画布图，不是 Dify DSL 的子集。22 种节点里真实在用的以 Python 代码节点为主，表达式是 JavaScript；「发布」只是元数据，保存即生效；人工输入、会话变量、文件传递用的都是 dsh 自己的机制。

控制面一侧：

- 转换器（画布图 → Workflow Definition）：映射表和 JS 的降级规则（D67，ADR-0015；`docs/open-issues.md` 的 OI-018）。
- 存量迁移：范围、工作流存放在哪、怎么打包进镜像，到迁移的时候定（OI-017）。引擎侧要能从部署包批量导入。
- 把已发布且非 hidden 的工作流注册成会话工具，工具名的派生规则不变。
- dsh 与引擎之间的文件搬运：调用前上传 file 型入参，调用后下载产出文件再声明交付卡。
- SSE 代理；人工输入的待答卡，能按 `session_ref` 在重启或重连后找回；暂停的界面。
- 待答卡显示全部字段（现有实现只渲染第一个字段，OI-041）；一个调用里同时有多张卡时逐张显示，同时只显示一张（D84）。
- 外部网关改调引擎，对外契约不变。
- 现有的两个 Python 第三方检索工具各自包成 HTTP 服务，注册成 Capability（Round 11 的默认处理第 1 条）；Markdown 转 docx 工具并进文档服务，作为「排版」接口的第一个实现（D82）。
- 画布保存只产生草稿，要发布才生效（D53）；撤回发布时下线引擎里的版本（D69）。
- 控制面保留「一个会话同时只有一个带未完成等待记录的调用」，按等待记录判断、不按调用状态（D72、D75）；一个调用里可以有多张卡，同时只显示一张（D84）。
- 画布的 `vars` 节点转成调用 dsh 会话变量接口的 http-tool 节点；dsh 侧要给引擎开一个带服务令牌的读写接口（D73、OI-063）。
- LLM 节点改为引擎直连模型，密钥走密钥库或网关（Round 10 的默认处理第 4 条）；dsh 侧工具超时的上限不再写死 900 秒（第 5 条）。
- 会话工具需要的两个引擎接口：按调用 id 取节点执行记录，按会话和工作流取最近一次成功的调用（Round 11 的默认处理第 2 条）。

**验收**（草案，等 OI-017 定了再确认）：

- 插件仓里 4 份在用的工作流快照经转换器进入引擎，转换报告没有阻断项，在引擎里跑通。
- 画布保存后不生效，发布后生效；撤回发布后引擎拒绝新的调用。
- 在会话里调用一个含人工输入的工作流：待答卡出现，提交后继续，完成后交付卡出现。
- 一个工作流的并行分支里两路同时在等人：同时只显示一张卡，提交后下一张才出现；其余在跑的分支没有被打断。
- 引擎重启之后，等待输入的待答卡能重新出现。
- 同一个会话里调用两个不同的工作流，两者并行，互不等待。

### M9 企业化验收

单实例压测（目标：单机 <1k 并发 Invocation）；准入控制与租户级上限生效；安全验收（鉴权、出口层、沙箱）；结构化日志和可选的 OpenTelemetry 追踪（Round 5 的默认处理）；部署物（引擎单文件、python-worker、迁移脚本、部署档位说明）。

**验收**：压测达标；超过上限返回 429。

## 3. 依赖顺序

M0 → M1 → M2 →（M3 与 M4 并行）→ M5 → M6 → M7 → M9。M8 独立推进。

## 4. 风险前置

| 风险 | 什么时候要确认 |
|---|---|
| 网关能否替换请求 body 里的占位符（OI-001） | M5 之前 |
| Dify 在 0.7.0 之后的新增节点与字段（F6） | M2、M4 的第一周 |
| 文档服务用什么语言实现（OI-003），以及选定的库的保真度（OI-059；验证清单已定，真实模板在另一台机器上，OI-125） | M6 之前 |
| 模板改了之后，正在用它的已发布工作流什么时候生效（OI-126） | M6 之前 |
| 转换器对存量画布图的覆盖率：有多大比例的节点要降级成代码节点（OI-018） | M8 的第一周 |
| dsh 与引擎的部署拓扑（OI-011） | M6 之前 |
| minijinja 与 Jinja2 的差异清单 | M4 的第一周 |
