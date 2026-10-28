# 部署

单二进制（`CGO_ENABLED=0`），支持 **linux/arm64 · armv7 · amd64** 与 **windows/amd64**——从树莓派 3B 到 x86 服务器同一份产物。默认端口 `:9080`。

## 安装方式一览

| 方式 | 适用 | 说明 |
|---|---|---|
| 源码构建 | 所有平台 | `git clone` + `make build`；web UI 已预构建内嵌，无需 node |
| systemd | Linux 常驻 | `deploy/mibee-bell.service` + `deploy/install.sh`（/opt/mibee-bell 布局） |
| Docker | NAS / 容器环境 | `deploy/compose.cn.yml`（国内拓扑）/ `compose.overseas.yml`（海外拓扑） |
| fnOS 应用 | 飞牛 NAS | `deploy/fnos/` 打包（fnOS 桌面集成，网关免登） |
| Windows | 桌面/服务器 | 交叉编译 `GOOS=windows GOARCH=amd64`，`mibee-bell.exe -config config.yaml` |

## systemd（推荐）

```bash
# 1) 构建/下载 linux 二进制到本机
make build

# 2) 安装到 /opt/mibee-bell + 装 unit + 建系统用户 mibee
sudo deploy/install.sh ./mibee-bell

# 3) 写配置（首次）
sudo $EDITOR /opt/mibee-bell/config.yaml
sudo /opt/mibee-bell/mibee-bell hash-password   # → auth.password_hash
sudo /opt/mibee-bell/mibee-bell encrypt-config  # → 渠道凭证加密迁移 crypto.key（可选）

# 4) 启动
sudo systemctl enable --now mibee-bell
journalctl -u mibee-bell -f
```

unit 自带加固：`ProtectSystem=strict`、`ReadWritePaths=/opt/mibee-bell`、`PrivateTmp`。升级 = 替换二进制（旧版自动留 `.bak` 时间戳备份）+ `systemctl restart`。

## Docker Compose

```bash
cd deploy
docker compose -f compose.cn.yml up -d
```

国内拓扑示例内置：Telegram 走渠道级出站代理；海外拓扑全渠道直连。数据卷挂载 SQLite 与 config.yaml。

## 树莓派 / 低配设备

- 实测单实例常驻 RSS 约 15–21MB；设计验收线树莓派 3B（512MB）< 80MB。
- armv7 / arm64 交叉编译：`CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./cmd/mibee-bell`。
- mesh 多节点天然适合"家庭主机 + 树莓派 + VPS"的异地混部（见 [高可用集群](mesh.md)）。

## 数据与备份

- SQLite 单文件（默认 `./mibee-bell.db`，WAL 模式）。备份：停机冷备或 `sqlite3 mibee-bell.db ".backup ..."`。
- 保留策略（每日自动清理）：events 30 天 / notifications 90 天 / deliveries 30 天 / feedback 永久（`storage.retention` 可调）。
- 迁移/灾备：`mibee-bell export` 反向导出规则与渠道 YAML。

## 反向代理

管理 UI 与 API 同端口，直接反代 `:9080` 即可；SSE（`/api/stream`）需要代理关闭缓冲（nginx: `proxy_buffering off`）。摄取端点可按需只暴露在内网。

## 监控 Bell 自己

- `/metrics`（Prometheus）+ `deploy/grafana-bell.json` 面板。
- 关键指标：`bell_events_dropped_total{stage}`（背压丢弃）、`bell_channel_health{channel}`（渠道降级）、`bell_escalation_pending`（挂起的升级）。
- 多节点部署时，node_down 死亡开关自动覆盖"Bell 自己挂了"的场景。
