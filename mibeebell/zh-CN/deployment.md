# 部署

单二进制零依赖，支持 **linux/amd64 · arm64 · armv6** 与 **windows/amd64**，另有**飞牛 fnOS 原生应用**（arm64 / x86）——从树莓派 3B 到 x86 服务器同一份产物。默认端口 `:9080`。

## 安装方式一览

| 方式 | 适用 | 说明 |
|---|---|---|
| 发布会下载 | 所有平台 | [发布仓 Release](https://github.com/mickeyzzc/releases/releases/tag/mibeebell-v0.1.0) 下载 tar.gz/zip + `checksums.txt` 校验 |
| fnOS 应用 | 飞牛 NAS | 下载 `.fpk`（arm64 / x86）→ 应用中心手动安装；桌面单点进入，网关免登 |
| systemd | Linux 常驻 | 下载二进制 + 下方 unit 示例（/opt/mibee-bell 布局） |
| Windows | 桌面/服务器 | 解压 `mibee-bell_windows_amd64.zip`，`mibee-bell.exe -config config.yaml` |
| Docker | 容器环境 | 规划中（v1.0 全量分发渠道），当前用二进制 + 容器内 systemd |

## systemd（推荐）

```bash
# 1) 下载并解压 linux 二进制
tar xzf mibee-bell_linux_amd64.tar.gz

# 2) 安装到 /opt/mibee-bell + 建系统用户 mibee
sudo useradd -r -s /usr/sbin/nologin mibee || true
sudo mkdir -p /opt/mibee-bell && sudo mv mibee-bell /opt/mibee-bell/

# 3) 写配置（首次）
sudo $EDITOR /opt/mibee-bell/config.yaml
sudo /opt/mibee-bell/mibee-bell hash-password   # → auth.password_hash
sudo /opt/mibee-bell/mibee-bell encrypt-config  # → 渠道凭证加密迁移 crypto.key（可选）

# 4) systemd unit（/etc/systemd/system/mibee-bell.service）
sudo tee /etc/systemd/system/mibee-bell.service <<'EOF'
[Unit]
Description=MiBeeBell alert notification hub
After=network-online.target

[Service]
User=mibee
WorkingDirectory=/opt/mibee-bell
ExecStart=/opt/mibee-bell/mibee-bell -config /opt/mibee-bell/config.yaml
Restart=always
RestartSec=3
# 加固
ProtectSystem=strict
ReadWritePaths=/opt/mibee-bell
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 5) 启动
sudo systemctl daemon-reload && sudo systemctl enable --now mibee-bell
journalctl -u mibee-bell -f
```

升级 = 下载新版二进制替换（旧版自动留 `.bak` 时间戳备份）+ `systemctl restart`。

## 树莓派 / 低配设备

- 实测单实例常驻 RSS 约 15–21MB；设计验收线树莓派 3B（512MB）< 80MB。
- 下载 `mibee-bell_linux_arm64.tar.gz`（树莓派级别）或 `armv6`（更老的 32 位板）。
- mesh 多节点天然适合"家庭主机 + 树莓派 + VPS"的异地混部（见 [高可用集群](mesh.md)）。

## 数据与备份

- SQLite 单文件（默认 `./mibee-bell.db`，WAL 模式）。备份：停机冷备或 `sqlite3 mibee-bell.db ".backup ..."`。
- 保留策略（每日自动清理）：events 30 天 / notifications 90 天 / deliveries 30 天 / feedback 永久（`storage.retention` 可调）。
- 迁移/灾备：`mibee-bell export` 反向导出规则与渠道 YAML。

## 反向代理

管理 UI 与 API 同端口，直接反代 `:9080` 即可；SSE（`/api/stream`）需要代理关闭缓冲（nginx: `proxy_buffering off`）。摄取端点可按需只暴露在内网。

## 监控 Bell 自己

- `/metrics`（Prometheus）——Grafana 面板可基于上述指标自建。
- 关键指标：`bell_events_dropped_total{stage}`（背压丢弃）、`bell_channel_health{channel}`（渠道降级）、`bell_escalation_pending`（挂起的升级）。
- 多节点部署时，node_down 死亡开关自动覆盖"Bell 自己挂了"的场景。
