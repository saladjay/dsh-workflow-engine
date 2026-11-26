# B. 数据库 Schema（修订稿，对应 D1–D84）

> 依据：D3、D7、D33–D38、D40、D43、D45、D46、D48、D50、D53–D58、D61、D63、D65、D68–D70、D73–D75、D77、D81、D82、D84，以及各轮的默认处理。决策原文见 `docs/grill-record.md`。
> 原则：utf8mb4；主键 UUID（引擎生成）；JSON 列用 MySQL `JSON`；跨服务不建物理外键；高频表按月分区预留。`tenant_id` 是环境变量配置的部署级固定值，引擎库里没有租户主数据（D68）。
> 执行用的配置只存引擎库，控制面经引擎的管理 API 读写（D34，ADR-0013）。引擎库和 dsh 自己的库分开，可以同实例，不建外键（Round 10 的默认处理）。
> 凡是「一次节点执行」都用 `node_run_id` 标识（D38）：迭代的一个元素、循环的一轮各算一次，重试算同一次。
> 留白的地方是实现期的细节，标了 OI 编号的见 `docs/open-issues.md`。

## 1. 工作流定义与版本（D33、D53、D54、D58）

```sql
workflows (
  id                 UUID PK,
  tenant_id          VARCHAR(64) NOT NULL,
  name               VARCHAR(255) NOT NULL,
  current_version_id UUID NULL,          -- 当前已发布的版本，同一时刻最多一个（D74）；为空表示没有，即从未发布，或者已下线
  created_at / updated_at
)

workflow_versions (
  id                 UUID PK,
  workflow_id        UUID NOT NULL,
  version            INT NOT NULL,       -- 递增
  definition         JSON NOT NULL,      -- Workflow Definition：Dify DSL 0.7.0 的结构 + 扩展命名空间
  source_dsl         MEDIUMTEXT NULL,    -- 原始的 Dify 导出文件，只有导入的版本才有
  status             ENUM('draft','published','unpublished') NOT NULL,
                                          -- published：当前生效的那个；unpublished：发布过、已下线（被新版本替换或被撤回）（D74）
  import_report      JSON NULL,          -- 阻断项与提示（错误级的导入已被拒绝，不会留在这里）
  model_mapping      JSON NULL,          -- 导入时用的模型映射规则和解析结果
  resolved_bindings  JSON NULL,          -- 发布时冻结的 Node Binding 解析结果
  compiled           JSON NULL,          -- 发布时生成的内部图表示，是定义的派生物
  published_at       DATETIME NULL,
  checksum           CHAR(64) NOT NULL,
  created_by VARCHAR(64), created_at,
  UNIQUE (workflow_id, version)
)
```

- 草稿可以原地更新；发布过的版本不再改变，要改就从发布过的版本派生新草稿（`POST /workflows/{id}/versions`），不带参数时取当前已发布的那个。一个工作流同一时刻只有一个草稿版本，已经有草稿时不能再新建。
- 环境变量在 `definition` 里，随版本一起冻结（D58）。导入时值为空的 secret 类型变量已经换成 `{{secret.变量名}}`。
- 一个工作流同一时刻只有一个已发布的版本（D74）。只有它能被调用：`POST /invocations` 不带版本号时取 `current_version_id`，带了版本号时必须是它，否则返回 `409`。幂等键先于版本校验。
- 发布新版本时，之前已发布的那个自动下线：它的 `status` 变成 `unpublished`，`current_version_id` 指向新的。在途的 Invocation 不受影响，照常用 `workflow_version_id` 记着的版本跑完。
- 下线只对当前已发布的版本有效：`status` 变成 `unpublished`，`current_version_id` 置空。下线过的版本可以重新发布，回滚就是这么做的：沿用当时冻结在 `resolved_bindings` 里的绑定，并且仍然校验；校验不过就拒绝发布（D69、D74）。
- 草稿可以改；发布过的版本（`published` 和 `unpublished`）内容不再变。

## 2. 执行记录（D36、D38、D54、D55）

```sql
invocations (
  id                    UUID PK,         -- invocation_id
  tenant_id             VARCHAR(64) NOT NULL,
  user_id               VARCHAR(64) NOT NULL,
  workflow_id           UUID NOT NULL,
  workflow_version_id   UUID NOT NULL,   -- 调用开始时的当前已发布版本；之后它被下线也不影响这次调用
  conversation_id       UUID NULL,       -- Conversation 绑定单一工作流；不传则没有会话变量和记忆
  session_ref           VARCHAR(128) NULL, -- 控制面会话的不透明引用：用于筛选、找回有未完成等待记录的调用，以及给 dsh 会话变量节点寻址（D73）
  parent_invocation_id  UUID NULL,       -- sub-workflow 派生关系
  parent_node_run_id    UUID NULL,       -- 父工作流里派生它的那次节点执行
  depth                 TINYINT NOT NULL DEFAULT 0,   -- 默认上限 5
  status                ENUM('pending','running','waiting_input','paused',
                             'succeeded','failed','cancelled') NOT NULL,
  idempotency_key       VARCHAR(128) NULL,
  idempotency_expires_at DATETIME NULL,  -- 保留期可配
  trigger_source        VARCHAR(32) NOT NULL DEFAULT 'api',
  inputs                JSON, outputs JSON, error JSON,
  config_snapshot       JSON NULL,       -- 调用开始时解析到的共享运行配置（Provider 配置、Capability、密钥的引用；
                                        -- gateway 档下没有真实值，真实值只在出口替换，ADR-0008）
  time_limit_ms         BIGINT NULL,     -- 整个工作流的执行时间上限
  running_time_ms       BIGINT NOT NULL DEFAULT 0,   -- 只累计状态为 running 的时间
  started_at / finished_at / created_at / updated_at,
  UNIQUE (tenant_id, workflow_id, idempotency_key),
  index: (tenant_id, created_at), (conversation_id), (parent_invocation_id), (status), (session_ref)
)

node_runs (
  id                   UUID PK,          -- node_run_id，跨重启稳定，重跑时沿用
  invocation_id        UUID NOT NULL,
  parent_node_run_id   UUID NULL,        -- 容器（迭代的一个元素、循环的一轮）本身也是一次节点执行
  node_id              VARCHAR(64) NOT NULL,
  node_type            VARCHAR(64) NOT NULL,
  attempt              INT NOT NULL DEFAULT 1,
  status               ENUM('running','succeeded','failed','skipped','waiting','cancelled','unknown') NOT NULL,
  inputs JSON, outputs JSON, error JSON,
  child_invocation_id  UUID NULL,        -- sub-workflow 节点派生的子 Invocation
  started_at / finished_at,
  index: (invocation_id, parent_node_run_id, node_id)
)

node_snapshots (                           -- 变量快照，只用于续跑
  id               BIGINT AUTO PK,
  invocation_id    UUID NOT NULL,
  node_run_id      UUID NOT NULL,         -- 这次快照是哪次节点执行之后的状态
  context          JSON NOT NULL,         -- 全量快照起步，优化期改增量 + 定期全量
  created_at,
  index: (invocation_id)
)

waiting_inputs (                           -- 人工输入的等待记录
  node_run_id     UUID PK,                -- 循环里每一轮的等待各是一条
  invocation_id   UUID NOT NULL,
  node_id         VARCHAR(64) NOT NULL,
  actions         JSON NOT NULL,          -- user_actions
  form_schema     JSON NULL,
  assignees       JSON NULL,              -- 可选，不透明的 user/role 引用
  remind_at       DATETIME NULL,          -- 提醒时限，可以不配
  reminded_at     DATETIME NULL,
  timeout_at      DATETIME NULL,          -- 超时时限，由引擎到期触发，走 __timeout 分支
  status          ENUM('waiting','resolved','timeout','cancelled') NOT NULL,
  resolved_by     VARCHAR(64) NULL, resolved_at DATETIME NULL,
  index: (invocation_id), (status, remind_at), (status, timeout_at)
)

events (                                   -- 只有结构性事件落库
  id             BIGINT AUTO PK,
  invocation_id  UUID NOT NULL,
  seq            BIGINT NOT NULL,         -- Invocation 内单调递增
  type           VARCHAR(32) NOT NULL,
  node_run_id    UUID NULL,
  payload        JSON,
  created_at,
  UNIQUE (invocation_id, seq)
)
```

- 节点开始时先写一行 `status = running` 再执行。进程重启后，`status = running` 的行就是在途节点，按配置的在途策略处理（本期：重跑，ADR-0011）。`status = waiting` 的行是挂在等待上的节点，保持等待，不重跑。
- 「在等人」按等待记录判断，不按调用状态（D75）：一路在等人、另一路在跑时，调用是 `running`，但 `waiting_inputs` 里有一条 `status = waiting` 的记录。找回待答卡、判断一个会话里有没有等人输入的调用（D72），都查 `waiting_inputs.status = 'waiting'`，可以经 `invocations.session_ref` 定位到会话。
- 一个会话同时只有一个等人输入的**调用**（D72）。一个**调用**里可以同时有多条等待记录（并行分支里两路各在等人）；同时显示给用户的只有一张卡，按等待记录产生的时间先产生的先显示，前一张被提交或超时之后才显示下一张（D84）。排队等显示的那些等待记录照常计各自的提醒时限和超时时限。
- 状态怎么取（D75）：只要有节点在执行，Invocation 就是 `running`；能执行的节点都执行完、只剩等人输入时，才是 `waiting_input`。每一路的等待各自是一条等待记录；没被等待挡住的分支照常推进。
- 汇合点什么时候开始（D75）：有两条以上入边的节点，要等每一条入边都「到齐」才开始。分两种情况：

  ```
          ┌──→ B（判断没选它，不会跑）──┐
    A ──→ 判断                           └──→ D（汇合点）
          └──→ C（判断选了它，在跑）  ───┘
  ```

  - 入边所在的分支**真的要跑**，比如 C 还在跑，或者在等确认卡：D 等它。
  - 入边所在的分支**根本不会跑**，比如 B 是判断没选中的那一路：它对 D 算「到齐」，D 不等它。不等的话 D 永远开始不了。
- 暂停期间，等待记录的 `remind_at`、`timeout_at` 停表；恢复时按暂停的时长顺延（Round 12 的默认处理）。
- 幂等键随节点执行下发：HTTP 出站请求带上由 `node_run_id` 派生的幂等头。
- `llm_delta`、`answer_delta` 不写 `events` 表（D37，ADR-0012）。序号来自同一个计数器，流式增量也占号；重启后从库里该 Invocation 的最大序号续（Round 12 的默认处理）。
- 重启时处于 `running` 的 Invocation 自动续跑；处于 `paused` 的保持暂停（D36、D49）。处于 `pending` 的重新入队；处于 `waiting_input` 的保持等待。重启后检查所有 `status = waiting` 的等待记录的时限，停机期间到点的按超时分支走，不管它所属的调用当时是什么状态（Round 12 的默认处理）。
- 子 Invocation 的截止时间不超过父 Invocation 的剩余时间（Round 5 的默认处理）。
- `tenant_id` 是部署级的固定值，由环境变量配置；`user_id` 是登录账号的 id（D68）。

## 3. 会话（D50、D56）

```sql
conversations (
  id           UUID PK,                   -- conversation_id，控制面按「Session × Workflow」分配
  tenant_id    VARCHAR(64) NOT NULL,
  workflow_id  UUID NOT NULL,             -- 一个 Conversation 只属于一个工作流
  user_id      VARCHAR(64) NOT NULL,
  session_ref  VARCHAR(128) NULL,         -- dsh Session 的不透明引用，只用于排查
  created_at
)

conversation_variables (
  conversation_id     UUID NOT NULL,
  name                VARCHAR(128) NOT NULL,
  value               JSON,
  updated_by_invocation UUID,
  updated_at,
  PK (conversation_id, name)
)

conversation_turns (                       -- 对话轮次；节点配置了 memory 才会被读取
  conversation_id UUID NOT NULL,
  invocation_id   UUID NOT NULL,
  turn_no         INT NOT NULL,
  query           MEDIUMTEXT,
  answer          MEDIUMTEXT,
  files           JSON NULL,              -- 这一轮涉及的 File 引用
  created_at,
  PK (conversation_id, invocation_id),
  index: (conversation_id, turn_no)
)
```

- 同一个 Conversation 里的 Invocation 串行，但只在工作流声明了会话变量时才排队。
- 会话变量在该 Conversation 第一次被使用时，按当时版本的声明初始化默认值；之后版本新增的会话变量，在第一次被引用时补默认值（Round 7 的默认处理）。
- `sys.dialogue_count` 取该 Conversation 里已有的调用数（Round 7 的默认处理）。

## 4. Capability、Provider 配置与密钥（D9、D35、D43）

```sql
capabilities (
  id          UUID PK,
  tenant_id   VARCHAR(64) NULL,           -- NULL = 全局能力
  name        VARCHAR(128) NOT NULL,
  version     INT NOT NULL,
  spec        JSON NOT NULL,              -- method / url / headers / auth / 超时 / 重试 / 输入输出 schema
                                          -- 密钥一律写 {{secret.name}}
                                          -- file_transfer: auto（缺省）| inline | presigned（D65）；
                                          -- 内联的阈值是引擎配置，默认 10MB
  enabled     BOOL DEFAULT TRUE,
  created_at / updated_at,
  UNIQUE (IFNULL(tenant_id,''), name, version)   -- tenant_id 可空；MySQL 的唯一键里 NULL 互不相等，所以包一层
)

provider_configs (
  id          UUID PK,
  tenant_id   VARCHAR(64) NULL,           -- NULL = 全局
  name        VARCHAR(128) NOT NULL,
  adapter     VARCHAR(64) NOT NULL,       -- 协议模式，如 openai-compatible
  base_url    VARCHAR(512) NOT NULL,
  api_key     VARCHAR(512) NOT NULL,      -- 一律是 {{secret.name}}
  models      JSON NOT NULL,              -- 可用模型与别名
  extra       JSON NULL,
  enabled     BOOL DEFAULT TRUE,
  created_at / updated_at,
  UNIQUE (IFNULL(tenant_id,''), name)              -- 同上
)

secrets (                                   -- 只在 local 部署档位存在
  id          UUID PK,
  tenant_id   VARCHAR(64) NOT NULL,
  name        VARCHAR(128) NOT NULL,
  ciphertext  VARBINARY NOT NULL,          -- AES-GCM
  key_version INT NOT NULL,
  created_at / updated_at,
  UNIQUE (tenant_id, name)
)
```

- `gateway` 档下引擎不持有密钥，没有 `secrets` 的内容（ADR-0008）。
- Capability 和 Provider 配置不随版本冻结，在调用开始时解析，结果记在 `invocations.config_snapshot`。
- dsh 会话变量不存引擎库：它由 dsh 保管，引擎经预置的 Capability（部署时写入的种子记录）调 dsh 的接口读写（D73）。

## 5. Node Binding（D54，ADR-0014）

```sql
node_bindings (
  id           UUID PK,
  tenant_id    VARCHAR(64) NOT NULL,
  scope        ENUM('type','instance') NOT NULL,
  node_type    VARCHAR(64) NULL,          -- scope=type 时必填
  workflow_id  UUID NULL,                 -- scope=instance 时必填
  node_id      VARCHAR(64) NULL,
  target       VARCHAR(255) NOT NULL,     -- 引擎里已注册的实现，或 capability:{id}
  config       JSON,
  created_at / updated_at,
  UNIQUE (tenant_id, scope, IFNULL(node_type,''), IFNULL(workflow_id,''), IFNULL(node_id,''))
)
```

这张表存的是配置。类型级绑定只对配置里不带目标地址的节点类型开放（knowledge-retrieval、document-extractor、code、template-transform、human-input）。发布时把解析结果写进 `workflow_versions.resolved_bindings`，之后改绑定不影响已发布的版本。

## 6. 文件（D57、D61、D65、D77）

```sql
files (
  id            UUID PK,
  tenant_id     VARCHAR(64) NOT NULL,
  invocation_id UUID NULL,                -- 由哪次调用产出；控制面上传的为空
  kind          ENUM('uploaded','produced','template') NOT NULL DEFAULT 'uploaded',
                                           -- template：文档模板，有名字，跨工作流复用，不随调用过期（D77）
  name          VARCHAR(255) NOT NULL,    -- 文件名；kind=template 时也是画布上显示的名字
  mime_type     VARCHAR(128) NOT NULL,
  size          BIGINT NOT NULL,
  storage_key   VARCHAR(512) NOT NULL,    -- 文件存储里的键，前缀按租户和 Invocation 划分
  sha256        CHAR(64) NULL,
  created_at,
  expires_at                              -- uploaded 和 produced 默认留 7 天；template 为空
)
index: (tenant_id, created_at), (invocation_id), (expires_at), (tenant_id, kind, name)
```

File 在变量里一律是引用：`{name, mime_type, size, url}`。文件存储是可插拔的（对象存储；本地开发用本地目录）。文件怎么在一次 HTTP 调用里进出，见 `docs/api-spec.md` 的第 5 节（D65）：小文件内联，大文件用预签名地址。预签名要求存储支持；本地目录的开发档位下，由引擎提供带令牌的上传和下载端点。

模板（`kind = template`）是文档服务做模板填充和「Markdown 排版成 Word」时用的版式文件（D77、D82）。它存在引擎这边，文档服务不自己做模板管理；调用时引擎按 D65 把它传给文档服务。同一个租户下模板按名字唯一。模板改了之后，正在用它的已发布工作流是立刻用新内容、还是要重新发布才生效，还没有定（`docs/open-issues.md` 的 OI-126）。

## 7. token 统计（D18）

```sql
token_usage (
  id                BIGINT AUTO PK,
  tenant_id         VARCHAR(64) NOT NULL,
  user_id           VARCHAR(64) NOT NULL,
  invocation_id     UUID NOT NULL,
  node_run_id       UUID NOT NULL,
  provider          VARCHAR(64) NOT NULL,  -- 解析后的 Provider 配置名
  model             VARCHAR(128) NOT NULL,
  prompt_tokens     BIGINT NOT NULL DEFAULT 0,
  completion_tokens BIGINT NOT NULL DEFAULT 0,
  total_tokens      BIGINT NOT NULL DEFAULT 0,
  raw_usage         JSON NULL,             -- 厂商返回的原始 usage
  latency_ms        INT,
  status            VARCHAR(16),
  created_at,
  index: (tenant_id, created_at), (invocation_id), (tenant_id, model, created_at)
)
```

不做成本换算。工作流的 token 由引擎单独记账，暂不并入 dsh 的会话归属账（D70）。

## 8. 留存（D63）

| 数据 | 默认保留 | 说明 |
|---|---|---|
| 变量快照 `node_snapshots` | Invocation 进入终态即删除 | 只用于续跑 |
| 调用记录、节点执行记录、事件、对话轮次 | 180 天 | 事后排查依据节点执行记录里的输入输出 |
| token 明细 `token_usage` | 2 年 | 财务依据 |
| 文件 `files` | 7 天 | 产出文件和控制面上传的入参文件用同一套留存（Round 12 的默认处理） |
| 幂等键 | 可配，到期清理 | |

带时长的三项（调用记录等、token 明细、文件）可按租户覆盖，覆盖值存在下面的表里，空的用默认值；变量快照在终态即删，幂等键的保留期是引擎配置。清理任务执行。

```sql
tenant_retention_policies (              -- 按租户覆盖留存期（Round 12 的默认处理）
  tenant_id              VARCHAR(64) PK,
  run_records_days       INT NULL,       -- 调用记录、节点执行记录、事件、对话轮次
  token_usage_days       INT NULL,
  files_days             INT NULL
)
```

## 9. 队列与实例（D46、D47）

- 内存队列，本期只支持单实例。多实例之前单独设计。
- 保证三条约束：Invocation 状态不只活在内存；广播是落库事件之上的优化；队列任务按 invocation_id 路由。
- 准入控制：运行中上限、队列长度上限（超限回 429）、可选的租户级并发上限。队列先进先出。
- 每个 Invocation 的节点执行总数有上限，是引擎配置项，默认值实现期对照 Dify 定（Round 7 的默认处理）。
- 重启续跑靠 `invocations.status`、`node_runs` 和 `node_snapshots` 重建，队列状态不落库；处于 `pending` 的调用重启后重新入队。
