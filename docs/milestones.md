# D. 实施里程碑（Rust crate 结构 + 阶段计划）

> 设计输入：grill 决策 D1–D32（见 docs/grill-record.md）、ADR-0001~0009、三份交付文档（映射表/schema/API）。

## 1. Cargo workspace 结构

```
engine/
├── crates/
│   ├── engine-model          # 图 IR、节点契约（类型化输入输出）、变量引用、事件枚举
│   ├── engine-core           # DAG 调度器（事件驱动/入度归零/skipped）、Invocation 状态机、
│   │                         # 变量上下文、Node Binding 路由、sub-workflow、队列抽象（内存→Redis 可换）
│   ├── engine-dify-import    # DSL 0.7.0 解析 → IR、静态引用校验、占位导入、导入报告
│   ├── engine-nodes          # 内置节点：if-else / aggregator / assigner / list-operator /
│   │                         # template-transform(minijinja) / iteration / loop / start / end / answer
│   ├── engine-llm            # Provider trait + OpenAI 兼容主干 + 适配器插槽（D5）、
│   │                         # 结构化输出分层管线（D6）、流式、token_usage 上报（D18）
│   ├── engine-capability     # Capability Registry 读取、http-tool/http-request 工厂、
│   │                         # 占位符透传（D24）、SSRF 黑白名单、热加载
│   ├── engine-humaninput     # waiting_inputs、唤醒事件、timeout 分支（D12）
│   ├── engine-code           # JS 池（rquickjs）+ Python worker 池管理（D30）
│   ├── engine-mcp            # MCP 客户端（tool+provider_type=mcp）
│   ├── engine-agent          # agent 节点基础循环（D27）
│   ├── engine-store          # 仓储 trait + MySQL(sqlx) 实现 + events 落库/回放
│   └── engine-api            # axum HTTP/SSE、鉴权中间件、API 校验
├── python-worker/            # Python worker 宿主（main shim + 长度前缀 JSON 协议）
├── migrations/               # sqlx 迁移（对应 docs/db-schema.md）
└── Cargo.toml                # workspace
```

**参考库使用约定**：`rig`（MIT/Apache）、`swiftide`（MIT/Apache）可按组件直接借用或参考；`graphbit` 为 **source-available 许可——只作架构参考，不复制代码、不直接引依赖**（法务确认前）；`pi_agent_rust` 同为参考。

## 2. 里程碑

### M0 地基（骨架可跑）
workspace 搭建、engine-model IR + engine-core 事件驱动 DAG 调度器 + 内存队列、engine-api 最小提交/查询、一个 echo 工作流端到端跑通（内存仓储）。
**验收**：提交→调度→完成→查状态，含 SSE 收到 invocation_finished。

### M1 执行核心（企业底座）
MySQL 仓储（migrations 落地）、invocation/node_runs/node_snapshots/events 全落库、取消/暂停/恢复/**进程重启续跑**、SSE 断线 Last-Event-ID 回放、幂等键。
**验收**：kill -9 引擎后重启，运行中 invocation 从快照续跑成功；Last-Event-ID 回放无缺事件。

### M2 LLM 线
Provider 主干（OpenAI 兼容端点配置驱动）+ 流式 llm_delta、结构化输出 D6 分层（原生 JSON mode → 提示词+JSON Schema 校验 → 错误注入重试 N 次）、token_usage 写入、llm/parameter-extractor/question-classifier 节点。
**验收**：Dashscope/DeepSeek/智谱 三厂商配置化接入；schema 校验失败自动重试可观测；token 明细可查。

### M3 Dify 兼容线
DSL 0.7.0 导入（解析→IR→静态校验→占位导入→报告）、节点语义对齐：if-else/aggregator（含 variable-assigner 别名归一）/assigner/iteration（并行+三种错误模式）/loop（三种终止）/list-operator/template-transform(minijinja)/end(output 别名)/answer 流式/trigger-* 转控制面配置。
**验收**：导入 2–3 个真实 Dify 导出 yml，import_report 准确，执行结果与 Dify 对照一致。

### M4 集成线
Capability Registry（CRUD+热加载）、http-tool + http-request、SSRF 黑白名单、`{{secret.*}}` 占位符透传、网关出口联调、human-input 全链路（挂起落库/唤醒/超时分支）、Conversation + conversation_variables + assigner、knowledge-retrieval（Capability 后端）。
**验收**：人工审批场景跨引擎重启完成；占位符请求经网关替换成功、替换失败被拒绝（ADR-0008）。

### M5 沙箱与扩展
Python worker 池（python-build-standalone + uv 预装 + 白名单 + seccomp/cgroups）+ JS 池 + code 节点、MCP 客户端、agent 基础循环、sub-workflow（深度上限）。
**验收**：Dify code 节点 yml 原样导入执行；恶意代码（死循环/超内存）被强杀不影响引擎。

### M6 企业化
Node Binding（类型默认+实例覆盖）落地、队列抽象换 Redis 多实例预留、性能压测（目标：单机 <1k 并发 invocation，D10）、安全加固验收（mTLS 预留）、部署物（单二进制 + python-worker + 迁移脚本 + 网关路由文档）。
**验收**：绑定 knowledge-retrieval→企业 KB 后，同一 DSL 兼容工作流语义不变；压测报告达标。

## 3. 依赖顺序

M0 → M1 → M3（兼容线可与 M2 并行）→ M4 → M5 → M6；M2 与 M3 并行时共享 M1 的事件/仓储底座。
风险前置项：M4 网关联调依赖企业网关侧排期 → 建议在 M1 期间即启动网关接口对齐；minijinja 与 Jinja2 的差异清单在 M3 首周产出。
