# B. 数据库 Schema 骨架（MySQL 优先，仓储接口抽象）

> 依据：D7（全量存储+每节点落库，ADR-0003）、D18（token 明细）、D20（Conversation）、D31（Node Binding）、ADR-0008（密钥两种模式）。
> 原则：字符集 utf8mb4；主键用 UUID（引擎生成）；JSON 列用 MySQL `JSON` 类型；跨服务不建物理外键（租户主数据在控制面库）；高频表（events、token_usage、node_runs）按月分区预留。
> 边界说明：工作流定义的**写入口**是控制面服务，但定义数据存引擎库（导入校验/静态引用校验在引擎），控制面经引擎管理 API 写入——单库部署简单，后续可拆。

## 1. 工作流定义

```sql
workflows (
  id            UUID PK,
  tenant_id     VARCHAR(64) NOT NULL,
  name          VARCHAR(255) NOT NULL,
  current_version_id UUID NULL,          -- 指向 workflow_versions
  created_at / updated_at
)
-- index: (tenant_id)

workflow_versions (
  id            UUID PK,
  workflow_id   UUID NOT NULL,
  version       INT NOT NULL,             -- 递增
  dsl_yaml      MEDIUMTEXT NOT NULL,      -- 原始 DSL（审计/重导入）
  graph_ir      JSON NOT NULL,            -- 解析后的内部图表示（导入产物）
  import_report JSON NOT NULL,            -- unsupported 清单、校验错误、绑定建议
  checksum      CHAR(64) NOT NULL,
  created_by    VARCHAR(64), created_at,
  UNIQUE (workflow_id, version)
)
```

## 2. 执行记录（每节点落库，ADR-0003）

```sql
invocations (
  id                 UUID PK,             -- = invocation_id
  tenant_id          VARCHAR(64) NOT NULL,
  user_id            VARCHAR(64) NOT NULL,
  conversation_id    UUID NULL,           -- chatflow 多轮时挂 Conversation
  workflow_id        UUID NOT NULL,
  workflow_version_id UUID NOT NULL,
  status ENUM('pending','running','waiting_input','paused',
              'succeeded','failed','cancelled') NOT NULL,
  idempotency_key    VARCHAR(128) NULL,   -- D3
  trigger_source     VARCHAR(32) NOT NULL DEFAULT 'api',  -- api | trigger 映射
  inputs             JSON, outputs        JSON, error JSON,
  parent_invocation_id UUID NULL,         -- sub-workflow 派生关系（D11）
  depth              TINYINT DEFAULT 0,   -- 子工作流深度（默认上限 5，D29）
  started_at / finished_at / created_at / updated_at,
  UNIQUE (tenant_id, workflow_id, idempotency_key),
  index: (tenant_id, created_at), (conversation_id), (parent_invocation_id), (status)
)

node_runs (
  id            UUID PK,
  invocation_id UUID NOT NULL,
  node_id       VARCHAR(64) NOT NULL,     -- Dify 节点 id 原样保留
  node_type     VARCHAR(64) NOT NULL,
  attempt       INT NOT NULL DEFAULT 1,   -- 重试次数（D19）
  status ENUM('running','succeeded','failed','skipped','waiting') NOT NULL,
  error         JSON, outputs JSON,
  started_at / finished_at,
  index: (invocation_id, node_id, attempt)
)

node_snapshots (                             -- 变量快照（断点续跑依据）
  id            BIGINT AUTO PK,
  invocation_id UUID NOT NULL,
  after_node_id VARCHAR(64) NOT NULL,      -- 该节点完成后的上下文快照
  context       JSON NOT NULL,             -- 全量快照起步；优化期改 增量+定期全量
  created_at,
  index: (invocation_id)
)

waiting_inputs (                             -- human-input 挂起（"待我审批"查询）
  invocation_id UUID NOT NULL,
  node_id       VARCHAR(64) NOT NULL,
  actions       JSON NOT NULL,             -- user_actions（对齐 Dify）
  form_schema   JSON NULL,
  timeout_at    DATETIME NULL,             -- __timeout 分支依据
  status        ENUM('waiting','resolved','timeout','cancelled') NOT NULL,
  resolved_by   VARCHAR(64) NULL, resolved_at DATETIME NULL,
  PK (invocation_id, node_id)
)

events (                                     -- SSE 事件落库（D13，Last-Event-ID 回放）
  id            BIGINT AUTO PK,
  invocation_id UUID NOT NULL,
  seq           BIGINT NOT NULL,            -- invocation 内单调递增
  type          VARCHAR(32) NOT NULL,       -- 11 类事件（见 api-spec.md）
  node_id       VARCHAR(64) NULL,
  payload       JSON,
  created_at,
  UNIQUE (invocation_id, seq)
)
```

## 3. 会话（D20）

```sql
conversations (
  id         UUID PK,                       -- conversation_id，控制面分配
  tenant_id  VARCHAR(64) NOT NULL,
  user_id    VARCHAR(64) NOT NULL,
  metadata   JSON, created_at
)
conversation_variables (
  conversation_id UUID NOT NULL,
  name            VARCHAR(128) NOT NULL,
  value           JSON,
  updated_by_invocation UUID,
  updated_at,
  PK (conversation_id, name)
)
```

## 4. Capability 与密钥（D9 / D17 / ADR-0008）

```sql
capabilities (
  id         UUID PK,
  tenant_id  VARCHAR(64) NULL,              -- NULL = 全局能力
  name       VARCHAR(128) NOT NULL,
  version    INT NOT NULL,
  spec       JSON NOT NULL,  -- method/url/headers 模板/auth 类型/超时/重试/输入输出 schema/{{secret.*}} 占位符
  enabled    BOOL DEFAULT TRUE,
  created_at / updated_at,
  UNIQUE (tenant_id, name, version)
)
capability_bindings (                       -- http-tool 节点 → capability 版本引用
  id UUID PK, tenant_id, node_ref VARCHAR(128), capability_id UUID, capability_version INT
)

secrets (                                    -- 仅"静态模式"；占位符模式的真值在网关侧
  id          UUID PK,
  tenant_id   VARCHAR(64) NOT NULL,
  name        VARCHAR(128) NOT NULL,
  ciphertext  VARBINARY NOT NULL,            -- AES-GCM（ADR-0008 两种模式并存）
  key_version INT NOT NULL,
  created_at / updated_at,
  UNIQUE (tenant_id, name)
)
```

## 5. Node Binding（D31，ADR-0009）

```sql
node_bindings (
  id            UUID PK,
  tenant_id     VARCHAR(64) NOT NULL,
  scope         ENUM('type','instance') NOT NULL,
  node_type     VARCHAR(64) NULL,           -- scope=type 时必填（如 knowledge-retrieval）
  workflow_id   UUID NULL,                  -- scope=instance 时必填
  node_id       VARCHAR(64) NULL,           -- scope=instance 时必填
  target        VARCHAR(255) NOT NULL,      -- builtin | capability:{id} | custom:{ref}
  config        JSON,
  created_at / updated_at,
  UNIQUE (tenant_id, scope, IFNULL(node_type,''), IFNULL(workflow_id,''), IFNULL(node_id,''))
)
```

## 6. token 统计（D18：只记 token 数 + model/provider）

```sql
token_usage (
  id              BIGINT AUTO PK,
  tenant_id       VARCHAR(64) NOT NULL,
  user_id         VARCHAR(64) NOT NULL,
  invocation_id   UUID NOT NULL,
  node_id         VARCHAR(64) NOT NULL,
  provider        VARCHAR(64) NOT NULL,
  model           VARCHAR(128) NOT NULL,
  prompt_tokens     BIGINT NOT NULL DEFAULT 0,
  completion_tokens BIGINT NOT NULL DEFAULT 0,
  total_tokens      BIGINT NOT NULL DEFAULT 0,
  latency_ms        INT,
  status            VARCHAR(16),
  created_at,
  index: (tenant_id, created_at), (invocation_id), (tenant_id, model, created_at)
  -- 成本 = 查询时按价格表版本换算，不在本表落库（D18）
)
```

## 7. 队列（D10）

内存队列为主实现；多实例时换 Redis Stream / MQ。队列状态**不落 MySQL**（执行时态在内存 + invocations.status 可重建），重启续跑靠 node_runs + node_snapshots 扫描 pending 节点。
