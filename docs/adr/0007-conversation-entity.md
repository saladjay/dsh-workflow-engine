# Conversation 实体与会话变量存引擎

引入 Conversation（会话）= 多个 Invocation 的容器；conversation_id 由控制面分配并传入，会话变量（conversation_variables）由引擎按 conversation_id 持久化，仅 assigner 类节点可写（对齐 Dify 语义）。会话变量是 Invocation 之间合法联动的受控通道——落实"保留隔离又能联动"的隔离设计。

## 修订

- 2026-11-26：Conversation 绑定单一 Workflow——conversation_id 由控制面按「dsh Session × Workflow」分配，会话变量按工作流声明。dsh 的 Session 是另一个概念：一个 Session 下 agent 可以调用多个工具，其中可以有多个工作流（grill Round 6 V2）。
- 2026-11-26：同一 Conversation 内的 Invocation 串行，但仅当工作流声明了会话变量；排队的范围是「同一会话里的同一个工作流」，agent 并行调用不同工作流互不阻塞。
- 2026-11-26：引擎为每个 Conversation 里的每次 Invocation 保留一个轮次（用户的查询 + answer 节点的回答文本）。节点（如 Dify 的 LLM 节点）配置了 memory 时，引擎按窗口取最近几轮拼进 prompt——这是 Dify 兼容的一部分，不是自动功能，节点没配 memory 就不碰历史。dsh 会话自己的记忆不归引擎（grill Round 6 V3、Round 7 W2）。
