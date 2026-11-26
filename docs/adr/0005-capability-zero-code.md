# Capability Registry 零代码节点派生

功能扩展走"注册而非编码"：http-tool 单一节点类型引用 Capability（HTTP API 定义：方法/地址/鉴权/schema/超时/重试），新增功能 = 注册一条记录 + 热加载，不改代码不发版。Dify 的 tool(provider_type=api) 节点直接映射 Capability；知识库等外部系统同样经 HTTP 能力对接。密钥一律用 `{{secret.*}}` 占位符引用，不在 Capability 里存明文或加密密钥（ADR-0008）。

## 修订

- 2026-11-26：原文「Capability 密钥 AES-GCM 加密落库」作废——密钥统一走占位符，AES-GCM 密钥库只在 local 部署档位存在（grill Round 5 U9）。
- 2026-11-26：Office 文档服务按本 ADR 接入——独立的 HTTP 服务，接口注册成 Capability，工作流里用 http-tool 节点引用；专用节点类型以后再加，届时绑定到这个 Capability（grill Round 9 Y1）。
- 2026-11-26：后端的响应和节点的输出契约不一致时，在后端前面放一个 HTTP 适配服务，Capability 指向适配服务，Capability 的定义里不做响应变换（grill Round 11 AA1）。
