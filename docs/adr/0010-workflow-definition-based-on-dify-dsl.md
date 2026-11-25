# Workflow Definition 以 Dify DSL 为底

引擎的原生工作流定义（Workflow Definition）直接采用 Dify DSL 0.7.0 的结构：所有 Dify 节点类型原样合法，自有节点类型（http-tool、sub-workflow 等）注册在引擎的节点类型表里，扩展字段放在保留的扩展命名空间中。Dify 导入因此是归一化、校验和出报告，而不是格式转换，且无损。graph_ir 这类内部图表示是每个版本的派生编译产物，不是定义的来源。

## Considered Options

- 独立 IR：引擎自有 schema，Dify DSL 与 dsh 画布各写一个编译器 → 否决：Dify 兼容已让 Dify 的结构成为必须支持的子集，平行再造一套只有成本，且导入不再无损
- 只把 Dify DSL 当导入格式、不定义原生格式（Round 1–4 的隐含状态）→ 否决：dsh 自己的工作流和自有节点没有进入引擎的途径

## Consequences

- 画布读写的是同一份定义；引擎对 Dify 节点配置的解析必须无损，否则画布打不开导入的工作流
- ADR-0002 中「自有扩展节点无法在 Dify DSL 中表达」的理由不再成立；「不承诺反向导出」的结论保留

## 修订

- 2026-11-26：引擎的定义写入接口只认 Workflow Definition（以及作为其来源之一的 Dify 导入），并提供新建版本、更新草稿、发布的 API；dsh 画布格式到 Workflow Definition 的转换属于控制面，不在引擎里（grill Round 7 W5）。
