# SRE 运维手册(SRE Handbook)

> 生产环境常用开源工具的 **下载 → 安装 → 部署 → 使用** 全流程技术文档,基于 Rocky Linux 9 / RHEL 9 实战整理。

## 📚 文档目录

### 📐 架构设计(architecture/)

> 围绕大型互联网 / 大型交易所 / 高并发·大流量·海量数据场景的架构梳理与组件选型。

| 文档 | 说明 |
|------|------|
| [01-高并发架构总览](architecture/01-高并发架构总览.md) | 分层架构全景、容量估算、演进路线、异地多活 |
| [02-大型交易所架构](architecture/02-大型交易所架构.md) | 撮合引擎、行情、清算账务、风控、冷热钱包 |
| [03-高并发流量治理](architecture/03-高并发流量治理.md) | 限流、熔断降级、削峰、秒杀、多级缓存 |
| [04-海量数据架构](architecture/04-海量数据架构.md) | 分库分表、NewSQL、实时数仓、数据湖 |
| [日志方案选型](architecture/selection/日志方案选型.md) | ELK vs Loki vs ClickHouse 系 vs VictoriaLogs |
| [监控方案选型](architecture/selection/监控方案选型.md) | Prometheus 生态、长期存储、链路追踪、告警体系 |
| [消息队列选型](architecture/selection/消息队列选型.md) | Kafka vs RocketMQ vs Pulsar vs RabbitMQ vs NATS |
| [数据库与缓存选型](architecture/selection/数据库与缓存选型.md) | MySQL/PG、NewSQL、Redis 及替代品 |
| [网关与负载均衡选型](architecture/selection/网关与负载均衡选型.md) | LVS、Nginx/OpenResty、Envoy、APISIX/Kong |

### 🗄️ 数据库(database/)
| 工具 | 说明 | 文档 |
|------|------|------|
| MySQL | 最流行的开源关系型数据库 | [database/mysql/](database/mysql/) |
| PostgreSQL | 功能最强的开源关系型数据库 | [database/postgresql/](database/postgresql/) |
| Redis | 高性能内存键值数据库/缓存 | [database/redis/](database/redis/) |
| MongoDB | 文档型 NoSQL 数据库 | [database/mongodb/](database/mongodb/) |

### 📈 监控(monitoring/)
| 工具 | 说明 | 文档 |
|------|------|------|
| Prometheus | 云原生时序监控系统 | [monitoring/prometheus/](monitoring/prometheus/) |
| Grafana | 可视化仪表盘平台 | [monitoring/grafana/](monitoring/grafana/) |
| Alertmanager | 告警路由与通知 | [monitoring/alertmanager/](monitoring/alertmanager/) |
| node_exporter | 主机指标采集器 | [monitoring/node_exporter/](monitoring/node_exporter/) |
| Zabbix | 传统企业级监控 | [monitoring/zabbix/](monitoring/zabbix/) |

### 📋 日志(logging/)
| 工具 | 说明 | 文档 |
|------|------|------|
| Elasticsearch | 分布式搜索与分析引擎 | [logging/elasticsearch/](logging/elasticsearch/) |
| Logstash | 日志采集与处理管道 | [logging/logstash/](logging/logstash/) |
| Kibana | ES 数据可视化 | [logging/kibana/](logging/kibana/) |
| Filebeat | 轻量级日志采集器 | [logging/filebeat/](logging/filebeat/) |
| Loki | 轻量级日志聚合系统(Grafana 生态) | [logging/loki/](logging/loki/) |

### 🔀 中间件(middleware/)
| 工具 | 说明 | 文档 |
|------|------|------|
| Nginx | 高性能 Web 服务器/反向代理 | [middleware/nginx/](middleware/nginx/) |
| Kafka | 分布式消息队列/流处理平台 | [middleware/kafka/](middleware/kafka/) |
| RabbitMQ | 企业级消息队列 | [middleware/rabbitmq/](middleware/rabbitmq/) |

### 📦 容器(container/)
| 工具 | 说明 | 文档 |
|------|------|------|
| Docker | 容器运行时 | [container/docker/](container/docker/) |
| Kubernetes | 容器编排平台 | [container/kubernetes/](container/kubernetes/) |

### 🤖 自动化(automation/)
| 工具 | 说明 | 文档 |
|------|------|------|
| Ansible | 无 Agent 批量配置管理 | [automation/ansible/](automation/ansible/) |
| Jenkins | CI/CD 流水线 | [automation/jenkins/](automation/jenkins/) |
| ccgram | Telegram ↔ tmux 桥接,手机远程控制 Claude Code | [automation/ccgram/ccgram部署与使用指南.md](automation/ccgram/ccgram部署与使用指南.md) |

### 🔐 安全(security/)
| 工具 | 说明 | 文档 |
|------|------|------|
| fail2ban | SSH 暴力破解防护 | [security/fail2ban/](security/fail2ban/) |

## 📖 文档规范

每个工具的文档统一按以下结构编写:

1. **简介** — 工具定位、适用场景、核心概念
2. **下载与安装** — 官方源 / 二进制 / 包管理器多种方式
3. **部署配置** — 生产级配置文件详解、systemd 托管
4. **使用指南** — 日常操作命令、常见运维场景
5. **故障排查** — 常见问题与解决方案

## 🖥️ 环境说明

- 操作系统:Rocky Linux 9.x(兼容 RHEL 9 / AlmaLinux 9 / CentOS Stream 9)
- 防火墙:firewalld
- SELinux:Enforcing
- 服务管理:systemd

## License

MIT
