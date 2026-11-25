# 触发器归控制面

DSL 的 trigger-schedule / trigger-webhook / trigger-plugin 节点映射为 dsh 控制面的触发配置；引擎只暴露统一的 API 触发入口，不内建调度器或 webhook 接收器。理由：触发入口唯一化，权限/审计/限流收口在控制面，引擎保持纯粹执行。
