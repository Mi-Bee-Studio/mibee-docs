# MiBeeNvr 集成

MiBeeNvr（开源 NVR）内置「告警通知」外发能力：把相机离线/画面冻结、存储健康、移动侦测、AI 事件、门铃、GB28181 告警等以 `mibee-alert/1` JSON 报文 POST 到 webhook 接收端。**MiBeeBell 是它开箱即用的接收端——零转换直连**。

```
NVR ──POST /api/ingest/webhook/{token}──▶ MiBeeBell 管道 ──▶ 渠道（企微/TG/邮件…）+ Scout
```

## 1. 注册摄取 token

`config.yaml` 给 NVR 专用 token（`source` 归因到 `mibee-nvr`，规则按它匹配）：

```yaml
ingest:
  webhook_tokens:
    - token: "nvr-ingest-9f3k2"     # 任意随机串，只存在 NVR 与 Bell 两侧
      default_source: "mibee-nvr"
```

NVR 侧（设置 → 通知）把接收端指向 Bell：

```yaml
notifications:
  enabled: true
  sinks:
    - name: "bell"
      url: "http://<bell-host>:9080/api/ingest/webhook"
      token: "nvr-ingest-9f3k2"
```

两侧 token 一致即通。

**测试告警的注意事项**（NVR 面板「发送测试告警」按钮发送 `notify.test` 类型，绕过 NVR 侧过滤直达，但 Bell 侧管道照常工作）：

- 测试告警默认 `warning` 级会被 **persist_filter** 吞（L0 持续过滤：10 分钟窗内第 3 次才放行）——首次联通验证请用 `error`/`critical` 级；
- 连续相似的测试会撞**相似聚合**与 **dedup**（所有 `notify.test` 共用同一去重键，60s 窗）——重复测试请换标题并间隔 60 秒以上；
- 想让 warning 级测试直达：给测试规则配 `persist_filter: 1`。

## 2. NVR 发什么

```json
{
  "schema": "mibee-alert/1",
  "event_id": "<uuid>",
  "occurred_at": "2026-10-25T04:05:06Z",
  "type": "camera.connection",
  "severity": "error",
  "title": "Camera offline: Studio",
  "source_ref": "cam-fa049182",
  "dedup_key": "mibee-nvr/connection/cam-fa049182",
  "resolved": false,
  "snapshot_url": "http://<nvr-host>:9090/api/ai/events/42/snapshot?api_key=mbv_...",
  "tags": { "camera_name": "Studio" }
}
```

Bell 的摄取映射：`type/severity/title/body/tags` 直传；`source_ref`（= 相机 ID，冷却与路由维度）直传；`dedup_key` 覆盖默认键；`resolved:true` 穿过去抖/冷却/静默并标记恢复。

**事件类型词汇表**（只增不改语义；未知类型走默认规则）：

| type | 说明 | 配对 |
|---|---|---|
| `camera.connection` | 离线 / 恢复 | 同 `dedup_key`，恢复侧 `resolved:true` |
| `camera.freeze` | 画面冻结 / 恢复 | 同上 |
| `camera.stream` / `camera.quality` | 码流异常 / 画质切换 | — |
| `camera.motion` | ONVIF 移动侦测（高频，靠管道聚合） | — |
| `ai.detection` | AI 事件（带 `snapshot_url`） | — |
| `doorbell.ring` | 门铃响铃 → 接听/拒接/挂断即恢复 | 同 `dedup_key` |
| `storage.health` | 存储故障 / 恢复 | 同 `dedup_key` |
| `gb28181.alarm` | 国标设备告警 | — |
| `camera.added` | 相机接入 | — |
| `notify.test` | NVR 测试按钮 | — |

**快照链接**：`snapshot_url` 是 NVR 侧可直接 GET 的图片 URL；AI/门铃帧不可变可缓存，`camera.motion` 的是实时帧。通知渠道与 Scout 均可直接展示。

## 3. 推荐规则包

按事件特性分级处理：状态型要冷却+升级链，高频型要聚合，恢复型要 `notify_resolved`。`CHANNEL` 换成你的渠道名。

**方式 A：REST 导入**（运行中的实例）：

```bash
BELL=http://192.0.2.10:9080
TOKEN=$(curl -s -X POST $BELL/api/v1/auth/login \
  -H 'Content-Type: application/json' -d '{"password":"<bell管理密码>"}' | sed 's/.*"token":"\([^"]*\)".*/\1/')
CH="chat"

post_rule() { curl -s -X POST $BELL/api/v1/rules -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d "$1"; echo; }

# 1) 相机离线：冷却 5 分钟；持续离线 15 分钟升级；恢复要通知
post_rule "{\"Name\":\"nvr-camera-offline\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.connection\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"EscalateAfterSeconds\":900,\"EscalateTo\":[\"critical\"],\"NotifyResolved\":true}"

# 2) 画面冻结：同款状态型处理
post_rule "{\"Name\":\"nvr-camera-freeze\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.freeze\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"NotifyResolved\":true}"

# 3) 移动侦测：高频——60s 去重 + 300s 冷却
post_rule "{\"Name\":\"nvr-motion\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"camera.motion\"},\"Channels\":[\"$CH\"],\"DedupSeconds\":60,\"CooldownSeconds\":300}"

# 4) AI 事件：60s 去重（快照在通知里）
post_rule "{\"Name\":\"nvr-ai\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"ai.detection\"},\"Channels\":[\"$CH\"],\"DedupSeconds\":60}"

# 5) 门铃：即时通知（无冷却）
post_rule "{\"Name\":\"nvr-doorbell\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"doorbell.ring\"},\"Channels\":[\"$CH\"],\"NotifyResolved\":true}"

# 6) 存储健康：严重且要升级链
post_rule "{\"Name\":\"nvr-storage\",\"Match\":{\"source\":\"mibee-nvr\",\"type\":\"storage.health\"},\"Channels\":[\"$CH\"],\"CooldownSeconds\":300,\"EscalateAfterSeconds\":900,\"EscalateTo\":[\"critical\"],\"NotifyResolved\":true}"
```

**方式 B：首启 seed**（全新实例，rules 表为空时导入一次）：

```yaml
rules:
  - name: nvr-camera-offline
    match: { source: mibee-nvr, type: camera.connection }
    channels: [chat]
    cooldown: 5m
    escalate_after: 15m
    escalate_to: [critical]
  - name: nvr-motion
    match: { source: mibee-nvr, type: camera.motion }
    channels: [chat]
    dedup: 60s
    cooldown: 5m
  - name: nvr-ai
    match: { source: mibee-nvr, type: ai.detection }
    channels: [chat]
    dedup: 60s
  - name: nvr-doorbell
    match: { source: mibee-nvr, type: doorbell.ring }
    channels: [chat]
  - name: nvr-storage
    match: { source: mibee-nvr, type: storage.health }
    channels: [chat]
    cooldown: 5m
    escalate_after: 15m
    escalate_to: [critical]
```

## 4. 调参建议

- **冷却/去重键**：冷却键是 `{rule}|{source_ref}`——同一台相机的离线和冻结互不干扰，这正是 `source_ref` = 相机 ID 的意义。
- **升级链与恢复**：`resolved` 事件穿透所有闸门；不想要"相机恢复了"的打扰，把对应规则 `notify_resolved` 关掉（REST POST 默认开）。
- **突发量**：NVR 侧已有 30 条/分钟限幅；`camera.motion` / `ai.detection` 成串时管道的聚合与去重窗口是主要防线。
- **夜间免打扰**（示例：移动侦测静默 8 小时）：

```bash
curl -s -X POST $BELL/api/v1/silences -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"match":{"source":"mibee-nvr","type":"camera.motion"},"duration":"8h","comment":"夜间免打扰"}'
```

- **验证**：`GET /api/v1/events?source=mibee-nvr` 看摄取侧、`GET /api/v1/deliveries?notification_id=` 看渠道侧；NVR 的 `GET /api/notifications/status` 有发送计数可对账。
