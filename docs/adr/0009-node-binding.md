# Node Binding：兼容节点的实现可切换

为 Dify DSL 兼容引入的节点类型保留"语义忠实实现"作为默认，同时允许控制面配置 Node Binding 把实现切换为自研后端（如 knowledge-retrieval → 企业知识库 Capability、http-request → http-tool）。粒度为类型级默认绑定 + 节点实例级覆盖；绑定实现必须满足原节点类型的输入输出契约——换实现不换语义，保住 DSL 兼容承诺（ADR-0002）。绑定关系与其他执行配置一样存引擎库，由控制面经引擎管理 API 读写，引擎按绑定路由执行（ADR-0013）。

## 修订

- 2026-11-26：原文「绑定关系存控制面配置」作废，与 api-spec.md、db-schema.md 的引擎库方案统一为引擎库单一存储（grill Round 5 U2）。
