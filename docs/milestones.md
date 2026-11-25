# D. 实施里程碑（修订稿，对应 D1–D63）

> 设计输入：grill 决策 D1–D63（见 `docs/grill-record.md`）、ADR-0001~0014。
> 和 Round 4 的初稿相比：Node Binding 的路由提前到地基里；新增「定义与发布」和「文件」两条线；补了 dsh 接入线；发布门槛和绑定冻结进了 M2。
> 标「待 Round 9」的地方等设计定了再补。

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

crate 的划分是实现期的建议，可以合并。Office 文档服务（C# 或别的语言）是这个 workspace 之外的独立服务，形态待 Round 9。

**参考库使用约定**：`rig`（MIT/Apache）、`swiftide`（MIT/Apache）可以按组件借用或参考；`graphbit` 是 source-available 许可，法务确认前只作架构参考，不复制代码、不直接引依赖；`pi_agent_rust` 同为参考。

## 2. 里程碑

### M0 地基

workspace 搭建；`engine-model` 解析 Workflow Definition；`engine-core` 的事件驱动调度器、Node Run 身份、Node Binding 路由的接口；内存队列；内存仓储；最小的提交与查询 API；一个 echo 工作流端到端跑通。

**验收**：提交、调度、完成、查状态，SSE 收到 `invocation_finished`。循环和并行迭代里，每一次节点执行都有各自的 `node_run_id`。

### M1 持久化与恢复

MySQL 仓储与迁移；事件只在节点边界落库，节点开始先落库再执行；流式增量只广播；取消、暂停、恢复；重启续跑（至少一次，幂等头随出站请求下发）；SSE 回放结构性事件；幂等键；准入控制；留存清理任务。

**验收**：`kill -9` 之后，运行中的 Invocation 从快照续跑成功，HTTP 后端凭幂等头没有重复执行；暂停中的保持暂停；回放不缺结构性事件。

### M2 定义与发布

定义 API（新建、更新草稿、发布）；Dify 导入（版本区间、归一化、节点类型校验、三级报告、模型映射、环境变量）；Provider 配置管理；Node Binding 的发布时冻结与契约校验。

**验收**：导入 0.1.5 到 0.7.0 的几份真实 DSL，报告的分级准确；带阻断项的版本发不了布；改了绑定，已发布版本的行为不变。

### M3 LLM 线

Provider 主干配置化接入；流式；结构化输出分层；`token_usage`；llm、parameter-extractor、question-classifier 节点；记忆（对话轮次）；Conversation、会话变量、会话内排队。

**验收**：三家国内厂商配置化接入；开了 memory 的节点能看到历史轮次，没开的完全不受影响；会话变量不丢更新。

### M4 Dify 兼容线

节点语义对齐：if-else、aggregator（含别名归一）、assigner、iteration（并行加三种错误模式）、loop（三种终止）、list-operator、template-transform、answer、trigger-* 转控制面配置、document-extractor 的内置格式。一致性夹具。

**验收**：对照夹具，执行结果与 Dify 一致。

### M5 集成线

Capability Registry；http-tool 与 http-request；出口层的两种档位与占位符；网关联调；SSRF 防护；MCP 远程客户端；human-input 全链路（两级时限、assignees、跨重启唤醒）；知识检索经绑定对接。

**验收**：审批场景跨引擎重启完成；占位符经网关替换成功，替换失败被拒；`local` 档下没有网关也能跑通。

### M6 文件线

文件存储接口；File 引用；产出入口（HTTP 二进制响应、code 节点、子工作流、生图）；与外部服务的文件交换（待 Round 9）；LLM 视觉输入内联；文档节点与服务（待 Round 9）；文件留存。

**验收**：工作流产出的 Word、Excel、PPT、图片能经签名链接下载；到期被清理。

### M7 沙箱与扩展

Python worker 池（python-build-standalone、uv 预装、白名单、seccomp / cgroups）；JS 池；code 节点；agent 基础循环；sub-workflow 的深度上限与不重复派生。

**验收**：Dify 的 code 节点原样导入执行；死循环或超内存的代码被强杀，不影响引擎。

### M8 dsh 接入线（在有 dsh 源码的机器上定）

控制面一侧：dsh 画布格式到 Workflow Definition 的转换、存量工作流的迁移策略、把已发布的工作流变成 agent 可见的工具、SSE 代理、人工输入与暂停的界面。

前置：查清 dsh 现有的工作流定义模型（F3）。

**验收**：待定。

### M9 企业化验收

单实例压测（目标：单机 <1k 并发 Invocation）；准入控制与租户级上限生效；安全验收（鉴权、出口层、沙箱）；部署物（引擎单文件、python-worker、迁移脚本、部署档位说明）。

**验收**：压测达标；超过上限返回 429。

## 3. 依赖顺序

M0 → M1 → M2 →（M3 与 M4 并行）→ M5 → M6 → M7 → M9。M8 独立推进，取决于 F3。

## 4. 风险前置

| 风险 | 什么时候要确认 |
|---|---|
| 网关能否替换请求 body 里的占位符（OI-001） | M5 之前 |
| Dify 在 0.7.0 之后的新增节点与字段（F6） | M2、M4 的第一周 |
| Rust 与 C# 的 Office 文档库各自的成熟度（F7、F8） | Round 9 的 Y4 |
| minijinja 与 Jinja2 的差异清单 | M4 的第一周 |
