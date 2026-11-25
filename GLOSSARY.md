# dsh Workflow 执行底层（企业版）

企业工作流平台执行底层的领域词汇表：dsh 控制面与 Rust 执行数据面共享的核心概念。

## Language

### 工作流定义

**Workflow Definition（工作流定义）**:
一个工作流「长什么样」的正式描述：图（节点与边）、变量声明与应用级特性，是画布编辑、引擎校验和执行的唯一依据。结构以 **Dify DSL** 为底，自有节点类型放在其上的扩展命名空间里。
_Avoid_: IR / graph_ir 充当「定义」（它们是定义的派生物）、原生 DSL、内部格式

**Workflow Version（工作流版本）**:
**Workflow Definition** 的一个具名快照，分草稿与已发布两种状态；已发布的版本不再改变，**Invocation** 总是针对某个已发布的版本执行。
_Avoid_: 用「工作流」指代某个具体版本

**Environment Variable（环境变量）**:
**Workflow Definition** 里声明的应用级只读变量，随 **Workflow Version** 一起冻结；它的值可以是一个 **Secret Placeholder**。
_Avoid_: 把它叫「配置」（易与 Provider 配置、Capability 混淆）

**Dify DSL**:
Dify 平台导出的工作流定义文件格式（.yml）；**Workflow Definition** 的来源之一。本引擎承诺可导入执行，不承诺反向导出。

### 执行与隔离

**Invocation（调用实例）**:
某 **Workflow Version** 的一次具体执行。引擎分配的 `invocation_id`（即 call-id）是其唯一标识与隔离边界：变量上下文、超时、取消、日志、token 统计均按 Invocation 切分。
_Avoid_: call、run、执行实例、session run

**Node Run（节点执行）**:
一个节点在一次 **Invocation** 的一个作用域里的一次执行：迭代的一个元素、循环的一轮各算一次，重试算同一次。容器（迭代、循环）本身也是一次节点执行，其内部节点的执行从属于它。
_Avoid_: 节点实例；用 (invocation_id, node_id) 或 attempt 充当执行身份

**Idempotency Key（幂等键）**:
上游业务传入的可选键；同一键的重复提交指向同一 Invocation，不产生新执行。

**Tenant（租户）**:
Invocation 的归属组织，隔离与统计的最高切分维度；其下还有人员维度 user_id。
_Avoid_: 用户（易与 user_id 混淆）

**Control Plane（控制面）**:
dsh（Node 企业版）：负责工作流定义、画布、触发入口、权限与结果查询；不执行工作流。

**Data Plane（数据面）**:
Rust 执行引擎：负责 Invocation 的调度与执行；不做权限与业务展示。
_Avoid_: 执行器（泛称，不作为组件名）

**Session（dsh 会话）**:
控制面里用户与 agent 的一次持续交互；agent 在其中可以调用多个工具，其中包括多个工作流。属于 **Control Plane**，不属于引擎。
_Avoid_: 把 Session 叫 Conversation

### 节点与集成

**Capability（能力）**:
一个已注册的 HTTP API 定义（方法/地址/鉴权/输入输出 schema/超时/重试），是 **Http-Tool Node** 可引用的功能单元。

**Capability Registry（端点注册表）**:
**Capability** 的注册与存储处；新增能力通过注册实现，零代码改动。

**Http-Tool Node**:
直接引用 **Capability** 的企业扩展节点；其他节点类型可经 **Node Binding** 以 Capability 为后端。
_Avoid_: 把经绑定走 Capability 的其他节点类型也叫 http-tool

**Provider Adapter（供应商适配器）**:
将某 LLM 云厂商接入统一主干协议的适配层；主流厂商走 OpenAI 兼容协议，特殊协议由 Adapter 转换。

**Provider Configuration（Provider 配置）**:
接入一家模型服务的一份具名配置（服务地址、凭据引用、可用模型）。**Workflow Definition** 里的模型引用经导入期映射指向它；与 **Provider Adapter**（协议转换）不是一回事。
_Avoid_: 模型配置（易与节点上的模型选择混淆）、把 Provider 配置叫 Adapter

**Conversation（会话）**:
同一个 **Workflow** 下多次调用（多轮对话）的容器；conversation_id 由控制面按「**Session** × Workflow」分配，会话变量（conversation_variables）跨 **Invocation** 持久。一个 Session 下每个工作流一个 Conversation。
_Avoid_: session（指 dsh 的会话时用 **Session**，是另一个概念）

**Turn（对话轮次）**:
**Conversation** 里的一次问答：对应一次 **Invocation** 的用户查询与回答文本。
_Avoid_: message、history（那是指 dsh 会话里的东西）

**Memory（记忆）**:
节点上可选开启的能力：把所属 **Conversation** 最近若干个 **Turn** 作为历史拼进提示词。节点没有配置它时完全不涉及历史。
_Avoid_: 把它和 dsh 会话的上下文混为一谈

**Sub-workflow（子工作流）**:
父 Invocation 经 sub-workflow 节点派生的子 Invocation；完成后结果回传父级，是跨 Invocation 数据交互的正规通道。

**Human-Input Node**:
暂停 **Invocation**、等待用户审批或表单输入后继续的节点；动作分支 + 超时分支，语义与 Dify human-input 对齐。

**Trigger（触发器）**:
工作流的入口事件（定时/webhook/插件事件）；归属 **Control Plane**，引擎仅接收统一 API 触发。

### 数据与文件

**File（文件）**:
工作流里按引用传递的二进制对象（文档、图片等）：变量里只放它的名称、类型、大小和地址，不放内容本身。
_Avoid_: 把文件内容直接放进变量；attachment（Dify 的变量类型叫 file / file-list）

**Secret Placeholder（密钥占位符）**:
以 `{{secret.name}}` 形式引用密钥的占位符，是配置中引用密钥的唯一写法。真实值只在请求发出的出口处替换；替换失败则请求被拒绝。
_Avoid_: xtoken（作通称）、与变量模板 `{{#...#}}` 混用、配置里写明文密钥

**Node Binding（节点绑定）**:
控制 **Dify DSL** 兼容节点实现来源的机制：类型级默认绑定 + 节点实例级覆盖；绑定实现必须满足原节点类型的输入输出契约——换实现不换语义。一份已发布的 **Workflow Version** 里每个节点用的是哪个实现是确定的，改绑定要重新发布才生效。
