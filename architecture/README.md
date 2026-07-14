# 架构设计(architecture/)

围绕**大型互联网 / 大型交易所 / 高并发·大流量·海量数据**场景的架构梳理与组件选型指南。

与其他目录的分工:其他目录(database/、monitoring/ 等)是单个工具的**部署实操文档**,本目录回答两个更上层的问题——**"架构长什么样"**和**"该选哪个"**。

## 📐 场景架构

| 文档 | 内容 |
|------|------|
| [01-高并发架构总览](01-高并发架构总览.md) | 大型互联网分层架构全景、容量估算方法论、架构演进路线、异地多活 |
| [02-大型交易所架构](02-大型交易所架构.md) | 撮合引擎、行情推送、清算账务、风控、冷热钱包、极端行情下的稳定性 |
| [03-高并发流量治理](03-高并发流量治理.md) | 限流、熔断降级、削峰填谷、秒杀架构、多级缓存、全链路压测 |
| [04-海量数据架构](04-海量数据架构.md) | 分库分表、NewSQL、冷热分离、实时数仓、数据湖、最终一致性 |

## 🔍 组件选型(selection/)

选型文档统一**结论先行**:开篇即"场景 → 推荐方案"速查表,再展开深度对比。

| 文档 | 覆盖组件 |
|------|----------|
| [日志方案选型](selection/日志方案选型.md) | ELK/EFK、Loki、ClickHouse 系、VictoriaLogs;Filebeat/Fluent Bit/Vector/iLogtail |
| [监控方案选型](selection/监控方案选型.md) | Prometheus、Thanos/VictoriaMetrics/Mimir、Zabbix/夜莺;Jaeger/Tempo/SkyWalking;告警体系 |
| [消息队列选型](selection/消息队列选型.md) | Kafka、RocketMQ、Pulsar、RabbitMQ、NATS |
| [数据库与缓存选型](selection/数据库与缓存选型.md) | MySQL/PostgreSQL、TiDB/OceanBase、Redis 及替代品、MongoDB/HBase/ES |
| [网关与负载均衡选型](selection/网关与负载均衡选型.md) | LVS/DPVS、Nginx/OpenResty、HAProxy、Envoy、APISIX/Kong、K8s Ingress |

## 阅读建议

- 做**方案设计/技术评审**:先读对应场景架构篇,再查选型篇的对比表。
- 做**面试准备**:01 → 03 → 04 按序读,交易所方向加读 02。
- 图表说明:全部架构图使用 Mermaid 绘制,GitHub / VS Code / mkdocs 原生渲染;量级数字均为经验参考值,以实际压测为准。
