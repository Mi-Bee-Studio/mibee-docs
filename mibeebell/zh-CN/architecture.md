# 架构

## 一句话架构

单进程 Go 二进制：**摄取层（七类入口）→ 事件总线（有界、满丢最旧）→ 告警管道（静默/去抖/冷却/升级 + L0 算法）→ 分发器（重试/限速/审计）→ 渠道插件（多渠道）**；SQLite 横切存储；REST + SSE + 嵌入式 Web UI 出口。

```mermaid
flowchart LR
    subgraph ingest[摄取层]
        A1[MQTT]
        A2[通用 Webhook]
        A3[Alertmanager / PagerDuty / Grafana / 简单 JSON / Bark]
    end
    A1 & A2 & A3 --> N[规范化 normalize]
    N --> BUS{{事件总线<br/>有界 channel · 满丢最旧<br/>永不阻塞发布者}}
    BUS --> P[管道：四道闸 + L0]
    P --> R[i18n 模板渲染<br/>zh-CN / en-US]
    R --> D[分发器<br/>重试 · 令牌桶 · 审计]
    D --> C[企微/钉钉/飞书 · SMTP · TG/Slack/Discord · Webhook · Scout]
    P --> DB[(SQLite WAL)]
    D --> DB
    DB --> E[REST /api/v1 · SSE /api/stream · /metrics · Web UI]
```

## 三条关键不变量

1. **发布者永不阻塞**：摄取回调只做规范化 + 非阻塞投递，总线满即丢最旧并计数（`bell_events_dropped_total{stage=bus}`）——**接口永不因下游故障 5xx**。
2. **恢复事件立即放行**：`resolved` 事件与故障侧同 `dedup_key` 配对成功后，跳过去抖/冷却（只走静默闸），并取消对应的升级计时。
3. **管道故障 ≠ 事件丢失**：任何管道环节 panic 由 recovery 兜底，事件按"无管道模式"直发默认渠道组并记审计事件。

## 背压模型

| 队列 | 默认容量 | 满时行为 |
|---|---|---|
| eventbus（摄取→管道） | 1024 | 丢最旧 + 计数 |
| dispatch queue（管道→渠道） | 每渠道 256 | 丢最旧 + 计数 |
| escalation 计时 | 持久化表 | 不丢（唯一持久化队列，进程重启后恢复计时） |

SQLite：WAL 模式、`busy_timeout=5s`、单写连接（写路径串行化）、读多连接。百条/天到万条/天量级富余。

## 存储可迁移

仓储只依赖 `database/sql`，SQL 全 ANSI（unix 毫秒整数时间戳、JSON 存 TEXT 应用侧编解码、`RETURNING id`、占位符统一 `?` 由 rebind 层转方言）——SQLite 方言特性只出现在连接器内。`storage.driver` 配置位已预留 PostgreSQL；迁移 = 实现 pg 连接器 + postgres 迁移目录，仓储与管道零改动（`migrations/postgres/` 与 sqlite 目录同 PR 维护）。

## 配置与运行态的边界

```
config.yaml             → 引导配置：server / auth / storage / ingest 连接
SQLite rules 表          → 路由规则（UI 可编辑）
SQLite channels 表       → 渠道配置（凭证加密落库，UI 可编辑）
config.yaml 的 rules/channels → 仅首次启动 seed（DB 为空时导入一次），此后 DB 为准
mibee-bell export        → 反向导出 YAML（备份/迁移）
```

这条边界让"改规则/换渠道"不需要碰配置文件，也让配置文件可以安全进版本管理（引导配置与凭证分离）。

## 包结构

```
MiBeeBell/
├── cmd/mibee-bell/       # 入口：flag、装配、优雅退出
├── internal/
│   ├── config/           # YAML 加载/校验 + hash-password / encrypt-config 子命令
│   ├── model/            # Event / Notification / Delivery / Rule / Channel / Silence
│   ├── eventbus/         # 有界总线（满丢最旧）
│   ├── mesh/             # memberlist 组网 + claim 协议 + 死亡开关
│   ├── ingest/           # 每源一个子包：mqtt / webhook / alertmanager / bark / …
│   ├── rule/             # 规则匹配（纯函数）
│   ├── pipeline/         # Processor 链 + 四道闸与 L0
│   ├── dispatch/         # 重试退避、令牌桶、投递审计、渠道健康
│   ├── channel/          # 渠道插件 + registry（自注册）
│   ├── ai/               # Analyzer（L1 决策 / L2 openai_compat）
│   ├── store/            # database/sql 仓储（sqlite/postgres 预留）+ 迁移
│   ├── render/           # i18n 模板（zh-CN/en-US，severity 映射）
│   ├── api/              # REST + SSE + JWT 中间件
│   └── ui/               # go:embed web/dist
├── pkg/bellapi/          # Channel / Processor / Analyzer 三接口——对外稳定契约
├── migrations/{sqlite,postgres}/
└── deploy/               # systemd / Docker compose / install.sh / fnOS / Zabbix
```

依赖方向单向：`ingest → eventbus → pipeline → dispatch → channel`；`model` 被所有人依赖、不依赖任何人。第三方扩展只 import `pkg/bellapi`（internal 禁止外部访问）。

## 可观测性

- `/metrics`（Prometheus）：`bell_events_ingested_total{source}`、`bell_notifications_total{channel,status}`、`bell_events_dropped_total{stage}`、`bell_channel_health{channel}`、`bell_escalation_pending`、`bell_pipeline_stage_duration_seconds`、`bell_ai_calls_total{analyzer,result}`、`bell_enrich_total`。
- 结构化日志（slog，JSON 可选）：丢弃 / 升级 / 静默命中 / 渠道降级等关键决策全部带 event_id。
- Grafana 面板 JSON 随仓库提供（`deploy/grafana-bell.json`）。

## 部署形态

单二进制（linux/arm64 · armv7 · amd64，windows/amd64）+ `deploy/` 提供 systemd unit、Docker compose（国内/海外两套拓扑示例）、install.sh、fnOS 应用打包与 Zabbix 媒体脚本。默认端口 `:9080`。
