# Open Issues（待补充事项）

> 记录已拍板决策里留给后续补充的事实。关闭时移入「已关闭」并写明依据。

## 进行中

### OI-001 网关能否替换 body 里的密钥占位符

- 状态：open（待补充）
- 来源：grill Round 5 U9 ②（2026-11-26）
- 问题：企业出口网关现在能否替换 HTTP 请求 **body** 中的 `{{secret.name}}`？JSON、multipart、流式三种 body 需分别确认。
- 已定的部分：密钥引用只有 `{{secret.name}}` 一种语法；出口按部署档位选 `gateway`（企业默认）或 `local`（ADR-0008）。
- 影响：网关不支持时，Round 3 S1 ② 的「body 允许占位符」收回，占位符只允许出现在 url、headers、auth 和环境变量里；`local` 档位不受影响。
- 待办：向网关侧确认能力与限制，结论写回 ADR-0008 与 docs/api-spec.md。

## 已关闭

（暂无）
