# 事件接入（七类入口）

面向接入方：任何监控系统、脚本、IoT 服务。摄取端点自 v0.1 起为**对外稳定契约**——字段只增不改语义。示例地址使用文档保留段 `192.0.2.10`。

## 入口总览

| 入口 | 端点 | 鉴权 | 适用 |
|---|---|---|---|
| **通用 webhook** | `POST /api/ingest/webhook/{token}` | URL 内 token（每 token 绑定一个 `source`） | 自研系统、脚本、任何能 POST JSON 的东西 |
| **Alertmanager 兼容** | `POST /api/ingest/alertmanager` | 内网暴露面（`ingest.alertmanager: true` 开启） | **Prometheus / Alertmanager 零改造** |
| **PagerDuty Events v2 兼容** | `POST /api/ingest/pagerduty` | `ingest.pagerduty: true` 开启 | 一切内置 "Send to PagerDuty" 的工具；Opsgenie 迁移落点 |
| **简单 JSON**（ntfy/Gotify 形） | `POST /api/ingest/simple` | `ingest.simple: true` 开启 | homelab 桥接生态零改造 |
| **Grafana 原生** | `POST /api/ingest/grafana` | `ingest.grafana: true` 开启 | Grafana 统一告警/旧版面板告警双形状，免配 Contact point 之外的任何东西 |
| **Bark 兼容路由** | `GET/POST /{key}`、`/{key}/{title}/{body}` | key 即渠道标识（`ingest.bark_compat: true`） | 存量 Bark 生态（NAS/Docker 通知脚本）改个域名迁入 |
| **MQTT** | 订阅 `{prefix}/health/+`、`{prefix}/event/+` | broker 账号 | MiBeeNVR 等生态设备直连 |

摄取成功 = **202 `{"event_id":...}`**——只代表已入总线，不代表已通知（通知结果看通知历史/投递记录）。摄取与分发异步，**接口永不因渠道故障 5xx**。

## 通用 webhook：Bell 信封契约

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/<你的token>" \
  -H "Content-Type: application/json" \
  -d '{"type":"device_lost","severity":"error","title":"客厅摄像头离线",
       "body":"NVR-01 已失联 10 分钟","source_ref":"cam-livingroom",
       "tags":{"site":"home"},"resolved":false,
       "dedup_key":"myapp/device_lost/cam-livingroom"}'
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `type` / `title` | 至少其一 | 事件类型 / 标题（title 缺省取 type） |
| `severity` | 否 | `info\|warning\|error\|critical`，缺省 warning；容忍业界同义词（crit/down/fatal/sev1…） |
| `body` | 否 | 正文 |
| `source_ref` | 否 | 事件主体标识（摄像头 ID / 设备 UUID——**冷却维度之一**） |
| `tags` | 否 | 任意 k/v，规则按子集匹配路由 |
| `resolved` | 否 | `true` = 恢复事件（跳过去抖/冷却、取消升级链） |
| `dedup_key` | 否 | 缺省自动 `source/type/source_ref`；**firing 与 resolved 用同一 key 即可配对关单** |
| `snapshot_url` | 否 | 快照图 URL，原样进通知（渠道/Scout 直接可拉） |

**多项目共用与路由隔离**：每个接入方一个独立 token，token 绑定 `source`（如 `mibee-steward` / `mibee-nvr`）。规则按 `source` 匹配路由——不同项目同机共用而互不串道（事件、静默、规则、升级全部按 source 分域）。

## 业界系统适配矩阵

| 系统 | 接法 | 改造成本 |
|---|---|---|
| Prometheus / Alertmanager | `webhook_configs` 指向 alertmanager 入口 | **零改造**（firing/resolved 自动配对） |
| Grafana 统一告警 | Contact point 选 Alertmanager 类型指向同一 URL，或用 grafana 原生入口 | **零改造** |
| 任意脚本 / cron / CI | curl POST 通用 webhook | 一行 |
| 内置 "Send to PagerDuty" 的工具（NetApp / OpenSearch / …） | 端点换成 pagerduty 入口 | **零改造** |
| Bark 生态存量脚本 | `api.day.app/yourkey` → `http://192.0.2.10:9080/yourkey` | 改个域名 |
| Zabbix 6+ | `deploy/zabbix/` 官方 media 脚本 | 一次配置 |
| Uptime Kuma / PRTG / 企微机器人格式等 | 薄适配层：十几行脚本把各家 JSON 映射成信封三件套再 POST | 一个转换脚本 |

薄适配示例（Uptime Kuma → Bell，Python 十行）：

```python
import json, urllib.request
def handler(payload):  # 挂在任意小服务/云函数上
    body = json.dumps({
        "type": "uptime_monitor", "severity": "error" if payload["heartbeat"]["status"] == 0 else "info",
        "title": f'{payload["monitor"]["name"]}: {"DOWN" if payload["heartbeat"]["status"] == 0 else "UP"}',
        "source_ref": payload["monitor"]["name"],
        "resolved": payload["heartbeat"]["status"] == 1,
    }).encode()
    urllib.request.urlopen(urllib.request.Request(
        "http://192.0.2.10:9080/api/ingest/webhook/<你的token>", body,
        {"Content-Type": "application/json"}))
```

> 适配层不需要部署在 Bell 旁——任何能同时访问源系统和 Bell 的机器均可；Bell 只认到达的信封。

## 端点细节速览

- **PagerDuty v2**：载荷对齐 `events.pagerduty.com/v2/enqueue`；`trigger/resolve` 同 `dedup_key` 配对，`acknowledge` 记为 info 级（可见不打扰）；响应带 PD 兼容 `{"status":"success","dedup_key":...}`。Bell 侧 `source` 固定 `pagerduty`。
- **简单 JSON**：正文取 `message/body/text` 之一；severity 优先级 = 显式 `severity` > `tags` 级别词（ntfy 惯例）> `priority`（0-3 info / 4-6 warning / 7-10 critical，或 low/default/high/urgent/max）> info；`topic` 可作 `source_ref` 载体。`source` 固定 `simple`。
- **Grafana**：统一告警形状（`status: firing|resolved` + `alertUid`）与旧版面板形状（`state: alerting|ok` + `ruleId`）都认；`ruleTags.severity` 优先。`source` 固定 `grafana`。
- **Bark 兼容**：sound/group/copy 等参数原样透传，未知参数忽略不报错；key 匹配渠道名或默认渠道组。
- **MQTT**：主题白名单（`health/*`、`camera.added`、`camera.quality`、`storage.health.changed`），断线自动重连；AI 侦测类事件走 NVR webhook 直投。

## 验证与排障

```bash
curl http://192.0.2.10:9080/healthz                # 存活
# 事件是否入库（登录后）：
curl -H "Authorization: Bearer $TOKEN" "http://192.0.2.10:9080/api/v1/events?source=<你的source>&limit=5"
```

令牌错了 → 404；格式错 → 400；进去了但没通知 → 查规则是否匹配该 source/type，以及是否被静默/L0 过滤（warning 级单次事件默认要连续 3 点才放行——测试请用 `error`/`critical` 级）。
