# API 参考

约定：除摄取端点与管理登录外，全部 `/api/v1/*` 需 `Authorization: Bearer <JWT>`。错误响应统一 `{"error":{"code":"...","message":"..."}}`；时间戳 unix 毫秒；列表按时间倒序，`?limit=` 取最近 N 条。

## 摄取端点（免 JWT）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/ingest/webhook/{token}` | 通用事件信封（字段见[事件接入](ingest.md)）；`source` 取 token 绑定值 |
| POST | `/api/ingest/alertmanager` | Alertmanager webhook JSON 原生格式 |
| POST | `/api/ingest/pagerduty` | PagerDuty Events v2 原生格式 |
| POST | `/api/ingest/simple` | 简单 JSON（ntfy/Gotify 形，字段宽容） |
| POST | `/api/ingest/grafana` | Grafana 统一告警 / 旧版面板告警双形状 |
| GET/POST | `/{key}`、`/{key}/{title}/{body}` | Bark 兼容路由（参数透传） |
| GET | `/healthz` | 存活 + 版本 |
| GET | `/metrics` | Prometheus（免 JWT，内网监听面） |

响应语义：摄取成功 = **202 `{"event_id":"..."}`**，只代表已入总线；摄取与分发异步，接口永不因渠道故障 5xx。

## 管理端点

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/v1/auth/login` | `{"password":"..."}` → `{"token":JWT,"stream_token":"..."}` |
| POST | `/api/v1/auth/stream-token` | 刷新 SSE 短票 |
| GET | `/api/v1/events` | `?since=&until=&source=&type=&severity=&dedup_key=&limit=` |
| GET | `/api/v1/notifications` | `?since=&severity=&limit=`；含聚合送达状态摘要 |
| GET | `/api/v1/deliveries?notification_id=` | 投递审计明细（每次尝试一行） |
| GET/POST/PUT/DELETE | `/api/v1/rules[/{id}]` | 规则 CRUD（match / channels / dedup / cooldown / escalate / persist_filter / notify_resolved / priority） |
| GET/POST/PUT/DELETE | `/api/v1/channels[/{id}]` | 渠道 CRUD；返回时凭证脱敏 `***` |
| POST | `/api/v1/channels/{name}/test` | 发测试通知（渲染 "MiBeeBell 测试" + 当前时间） |
| GET/POST/DELETE | `/api/v1/silences[/{id}]` | 静默窗口。POST 支持：绝对窗口、`{"dedup_key":"...","duration":"1h"}` 一键静默、`{"cron":"23:00-06:00"}` **每日时间窗**（支持跨午夜） |
| POST | `/api/v1/events/{id}/feedback` | `{"action":"confirm\|false_positive\|ack"}` |
| GET | `/api/v1/mesh` | 集群观测（成员/状态/claim/静默同步） |
| GET | `/api/v1/extensions` | 已注册渠道/处理器/分析器枚举（可审计的扩展性） |

## SSE 流

```
GET /api/stream?token=<stream_token>&since=<unix-ms>
```

- `stream_token`：登录返回，5 分钟有效可刷新（EventSource 无法带 header，故独立短票）。
- 事件类型（SSE `event:` 字段）：
  - `notification` —— 新通知（title/body/severity/snapshot_url；Scout 前台消费同一流）
  - `delivery` —— 送达状态变更（渠道 ok/degraded）
  - `silence` / `escalation` —— 静默生效 / 升级触发（UI 实时徽标）
- 断线重连：客户端带 `Last-Event-ID` **请求头**续传；服务端内存环形缓冲最近 512 条。

## Scout 通知中心 API

面向 MiBeeScout App 的稳定契约（M8 已实现）：

- `GET /api/v1/notifications?since=&unread=1` —— 分页拉取；`unread=1` 只返回未读；行内含 `SnapshotURL` / `Source` / `SourceRef`（**稳定承诺字段**）与 `ReadAt`（null = 未读）。
- `POST /api/v1/notifications/{id}/read` —— 标记已读（幂等；未知 id 404）。
- SSE `notification` 事件与上述行字段一致（同一结构序列化）。
- Scout 深链协议（点击跳 NVR 回放）在 Scout 侧定义；Bell 只保证 `snapshot_url` 与事件源引用字段稳定。

## 兼容性承诺

- **摄取端点从 v0.1 起视为对外稳定契约**：字段只增不改语义。
- `/api/v1/*` 在 v0.x 阶段允许破坏性调整，v1.0 起冻结。
