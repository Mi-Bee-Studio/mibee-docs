# 快速开始

目标：**5 分钟内**在一台 Linux 机器上跑起 MiBeeBell，发出第一条告警并收到通知。

> 命令逐字可执行。示例地址使用文档保留段 `192.0.2.10`，替换为你的实例地址。

## 1. 下载与安装

从「[发布会](https://github.com/mickeyzzc/releases/releases/tag/mibeebell-v0.1.0)」下载对应平台的制品并校验（附 `checksums.txt`）：

| 文件 | 适用 |
|---|---|
| `mibee-bell_linux_amd64.tar.gz` | Linux x86_64 |
| `mibee-bell_linux_arm64.tar.gz` | Linux ARM64（树莓派级别） |
| `mibee-bell_linux_armv6.tar.gz` | Linux ARMv6 |
| `mibee-bell_windows_amd64.zip` | Windows x86_64 |
| `mibee-bell-fnos-0.1.0-arm64.fpk` / `-amd64.fpk` | 飞牛 fnOS NAS（应用中心手动安装） |

```bash
tar xzf mibee-bell_linux_amd64.tar.gz && chmod +x mibee-bell
```

## 2. 首次启动：设置管理密码

直接运行（配置文件同目录 `config.yaml`，最小配置即可）：

```bash
cat > config.yaml <<'EOF'
server:
  listen: ":9080"

ingest:
  webhook_tokens:
    - token: "my-ingest-token-1"      # 任意长随机串：接入方与 Bell 各存一份
      default_source: "myapp"
  alertmanager: true                  # 顺手打开 Prometheus/Grafana 兼容入口
EOF

./mibee-bell
# log: mibee-bell listening addr=:9080 ...
```

浏览器打开 `http://192.0.2.10:9080`，首次进入是「**设置管理员密码**」页面——设置后自动登录，进入通知中心。（fnOS 应用形态下，飞牛管理员经桌面单点进入，无需二次登录。）

也可以走配置文件方式：`./mibee-bell hash-password --password '你的管理密码'`，把输出的 `password_hash` 填到 `auth.password_hash`。

> 管理密码未设置时管理 API 与 Web UI 停用（日志警告）；摄取入口不受影响。

## 3. 启动

```bash
./mibee-bell -config config.yaml
# log: mibee-bell listening addr=:9080 ... api=true
curl http://127.0.0.1:9080/healthz
# {"status":"ok","service":"mibee-bell",...}
```

浏览器打开 `http://192.0.2.10:9080`，用管理密码登录，进入通知中心。

## 4. 配一个渠道

UI 左侧「通知渠道」→ 新建。最简单的验证渠道是**通用 Webhook**（填任意能收 POST 的 URL）；正式使用填你的群机器人：

| 渠道 | 关键凭证 |
|---|---|
| 企业微信机器人 | 群机器人 Webhook URL 里的 `key` |
| 钉钉机器人 | `access_token` + 加签 `secret` |
| 飞书机器人 | Webhook URL（可选签名校验） |
| Telegram | Bot `token` + `chat_id`（可配渠道级代理） |
| SMTP | 主机 / 账号 / 密码 |
| 通用 Webhook | 目标 URL（可选 HMAC-SHA256 签名） |

配完点「发送测试」验证连通。凭证落库自动 AES-256-GCM 加密，界面/API 永远只显示 `***`。

## 5. 建一条规则

UI「路由规则」→ 新建：

```yaml
name: myapp-errors
match:
  source: myapp            # 匹配第 2 步 token 绑定的 source
  severity_gte: error
channels: [你的渠道名]
cooldown: 5m               # 同一主体 5 分钟内不重复轰炸
escalate_after: 15m        # 15 分钟没恢复 → 升级重发
```

## 6. 发第一条告警

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/my-ingest-token-1" \
  -H "Content-Type: application/json" \
  -d '{"type":"demo_alert","severity":"error","title":"第一条测试告警",
       "body":"hello from curl","source_ref":"demo-1"}'
# 202 {"event_id":"..."} —— 已入管道；通知结果看 UI 通知中心与投递记录
```

再发一条同 `dedup_key` 的**恢复事件**，观察"已恢复"通知自动配对送达：

```bash
curl -X POST "http://192.0.2.10:9080/api/ingest/webhook/my-ingest-token-1" \
  -H "Content-Type: application/json" \
  -d '{"type":"demo_alert","severity":"info","title":"第一条测试告警已恢复",
       "source_ref":"demo-1","resolved":true}'
```

## 7. 接入现有系统（零改造）

```yaml
# Prometheus Alertmanager —— alertmanager.yml
route:
  receivers: [bell]
receivers:
  - name: bell
    webhook_configs:
      - url: http://192.0.2.10:9080/api/ingest/alertmanager
```

Bark 生态存量脚本只需把 `api.day.app/yourkey` 换成 `http://192.0.2.10:9080/yourkey`。更多入口见[事件接入](ingest.md)。

## 要求与下一步

- 任意 Linux（arm64 / armv7 / amd64）或 Windows x64；内存 512MB 起步足够。
- 下一步：[Web 界面巡礼](web-ui.md) · [告警管道](pipeline.md) · [高可用集群](mesh.md)
