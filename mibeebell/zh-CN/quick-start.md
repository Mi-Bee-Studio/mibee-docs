# 快速开始

目标：**5 分钟内**在一台 Linux 机器上跑起 MiBeeBell，发出第一条告警并收到通知。

> 命令逐字可执行。示例地址使用文档保留段 `192.0.2.10`，替换为你的实例地址。

## 1. 构建与安装

从源码构建（web UI 已预构建内嵌，无需 node）：

```bash
git clone https://github.com/Mi-Bee-Studio/MiBeeBell.git
cd MiBeeBell
make build          # 产出 ./mibee-bell（CGO_ENABLED=0，交叉编译友好）
```

系统级安装（可选，systemd 托管 + `/opt/mibee-bell` 布局）：

```bash
sudo deploy/install.sh ./mibee-bell
```

## 2. 写一份最小配置

```bash
./mibee-bell hash-password --password '你的管理密码'
# 输出 {"password_hash":"$2a$10$..."}，填到下面 auth.password_hash

cat > config.yaml <<'EOF'
server:
  listen: ":9080"

auth:
  password_hash: "$2a$10$..."        # 上一步的输出

storage:
  path: ./mibee-bell.db

ingest:
  webhook_tokens:
    - token: "my-ingest-token-1"      # 任意长随机串：接入方与 Bell 各存一份
      default_source: "myapp"
  alertmanager: true                  # 顺手打开 Prometheus/Grafana 兼容入口
EOF
```

> `auth.password_hash` 为空时管理 API 与 Web UI 停用（日志警告）；摄取入口不受影响。

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
