# 通知渠道

原则（设计铁律 3）：**一个渠道一个文件一个接口**。渠道平台政策变化，死的只是那个适配器。适配器只做"渲染后的消息 → 该平台 API 调用"；重试、限速、审计、健康探测全部集中在分发器。

## 内置渠道

| type | 渠道 | 自报限速（默认） | 备注 |
|---|---|---|---|
| `wecom_robot` | 企业微信群机器人 | 18 / 60s | 官方 20/min，留安全余量 |
| `dingtalk_robot` | 钉钉自定义机器人 | 18 / 60s | 加签（secret + timestamp） |
| `feishu_robot` | 飞书自定义机器人 | 18 / 60s | 支持签名校验（可选） |
| `smtp` | SMTP 邮件 | 6 / 60s | QQ/163/SES/Resend 通吃；正文 Markdown→HTML |
| `telegram` | Telegram Bot | 20 / 60s | 支持 per-channel 出站代理（socks5/http） |
| `slack` | Slack Incoming Webhook | 1 / 1s | |
| `discord` | Discord Webhook | 5 / 60s | 官方 50/min |
| `webhook` | 通用 Webhook | 10 / 60s | JSON POST + 可选 HMAC-SHA256 签名头 |
| `scout` | MiBeeScout 通知中心 | — | 事件落库 + 通知中心 API/SSE 消费 |
| `bark` | Bark（iOS 推送） | — | 服务器 + 设备 key；兼容路由另支持存量 Bark 脚本零改造迁入 |
| `ntfy` | ntfy | — | 官方 ntfy.sh 或自建；主题订阅 |
| `pushover` | PushOver | — | token + user |
| `wecom_app` | 企业微信应用消息 | — | corpid + 应用凭据（走应用消息而非群机器人） |
| `twilio` | Twilio 语音/短信 | — | 付费渠道：仅允许出现在 `escalate_to` 升级链，受日预算熔断保护 |
| `pagerduty` | PagerDuty Events v2 | — | 付费渠道：同上 |
| `opsgenie` | Opsgenie | — | 付费渠道：同上 |
| `line` | LINE Messaging API | — | channel token |

共 **17 类**内置渠道。长尾渠道经通用 `webhook` 适配层兜底。

## 渠道配置

UI「通知渠道」页或 `POST /api/v1/channels` 配置，每渠道独立：

- **凭证字段**（key/token/secret/webhook url）——写时 AES-256-GCM 加密落库（`enc:` 前缀），读时解密给适配器，界面/API 输出一律 `***` 脱敏。
- **`proxy`**：渠道级出站代理——国内实例发 Telegram 走本机 socks5，互不影响其他渠道。
- **`timeout`**：默认 10s。
- **启停与健康**：连续 5 次投递失败 → `health_status=degraded` → UI 标记 + 可选通知管理员"该换渠道了"。建议默认配置永远 ≥2 渠道。

```yaml
# config.yaml 首启 seed 示例（之后以 DB 为准，UI 可编辑）
channels:
  wecom-ops: { type: wecom_robot, key: "enc:..." }
  tg:        { type: telegram, token: "enc:...", chat_id: "123", proxy: "socks5://127.0.0.1:1080" }
```

## 分发器职责（适配器不管的事）

1. **重试**：指数退避 1s/2s/4s/8s/16s，最多 5 次；4xx（除 429）不重试——配置错误重试也没用。
2. **限速**：per-channel 令牌桶（渠道自报限速与配置取较小值），事件风暴下排队不阻塞。
3. **审计**：每次尝试落 `deliveries` 表（状态 / 尝试次数 / 响应码 / 错误摘要），UI 投递记录可查。
4. **日预算**：付费渠道按 `daily_budget_events` 熔断——触顶跳过并记审计，不阻塞其他渠道。

## 消息渲染

- 模板矩阵：`(severity × channel_family × lang)`，内嵌 zh-CN / en-US 双语，缺省回落 en-US。
- 渠道家族差异：IM 类用各自 Markdown 方言（企微 markdown / 钉钉 markdown / 飞书 post / TG HTML）；SMTP 用 HTML；通用 webhook 输出结构化 JSON（含全部事件字段）。
- 通知必含：severity 图标 + 标题、正文、快照缩略图（`snapshot_url` 存在且渠道支持时）、RFC 3339 时间戳（本地时区）、来源标识（MiBeeBell footer + 规则名）。
- 聚合计数（×N）与"已升级"标记在渲染层注入正文。

## 渠道适配机制

- 渠道是一个个独立适配器：**一个渠道一个适配器**，新增渠道不改动核心；
- 在 DB（UI）或 config 加一段配置即可启用一个新渠道实例；
- 渠道平台政策变化只影响它自己的适配器，其余渠道不受影响。

需要新渠道适配？欢迎在[发布会仓库](https://github.com/mickeyzzc/releases/issues)提 issue 说明目标平台与凭证形态。
