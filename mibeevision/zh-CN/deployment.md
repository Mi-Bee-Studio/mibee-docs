# 部署与快速开始

MiBeeVision Lite 以单二进制分发，从「**发布会**」（[mickeyzzc/releases](https://github.com/mickeyzzc/releases)）下载，免费使用。

## 下载

发布地址：[发布会 · MiBeeVision Lite v0.1.0](https://github.com/mickeyzzc/releases/releases/tag/mibee-vision-v0.1.0)

| 资产 | 说明 |
|---|---|
| `mibee-vision-lite-v0.1.0-linux-x86_64` | x86_64 二进制（内置 YOLOv8n 模型） |
| `mibee-vision-lite-v0.1.0-linux-aarch64` | ARM64 二进制（内置 YOLOv8n 模型） |
| `libonnxruntime-1.17.1-linux-x64.so` | ONNX Runtime 运行时（CPU，x86_64） |
| `libonnxruntime-1.17.1-linux-aarch64.so` | ONNX Runtime 运行时（CPU，ARM64） |
| `SHA256SUMS` | 校验和 |
| `README.txt` | 随包说明 |

- 平台：**Linux x86_64 / aarch64**（不提供 Windows / macOS 版本）。
- Lite 限制：单摄像头、CPU 推理；多摄像头 / GPU 加速 / 历史批量补分析见 Pro 版（规划中，发布时间待公布）。

## 两个必读前置

1. **必须设置 `ORT_DYLIB_PATH`** 指向随包的 `libonnxruntime-*.so`。缺失时推理库加载失败，管线无法处理录像段——症状是「配置页能打开，但永远不出事件」。
2. **必须使用 NVR 的 `mbv_` API Key**（NVR「设置 → AI 检测 → 生成 API Key」），不要用网页登录 token（`mbs_` 开头）——后者回写 AI 事件会返回 401。

详见 [MiBeeNvr 协同](nvr-integration.md)。

## 快速开始（以 ARM64 为例）

```bash
# 1. 解包二进制与对应架构的 .so 到同一目录
sudo mkdir -p /opt/mibee-vision
# （把下载的 4 个文件放入 /opt/mibee-vision/）

# 2. 校验完整性（可选但建议）
sha256sum -c SHA256SUMS

# 3. 运行（ORT_DYLIB_PATH 不可省略）
ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so \
  /opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve

# 4. 浏览器打开 http://<本机IP>:9092
#    填 NVR 地址 → 粘贴 mbv_ API Key → 「测试连接并拉取摄像头」
#    → 选择 1 个摄像头 → 填规则名与区域顶点 → 「保存(自动重启管线)」
```

**自检**：配置页状态区应显示「管线 运行中 / NVR 已连接」；几分钟后 NVR 录像列表的 AI 状态列出现 `completed`，即全链路打通。

## 跨机部署

Vision 不在 NVR 主机上时，传输方式（`[transport].type`）必须选 **`http_download`**（配置页中设置，保存自动生效）；`local_file` 模式要求与 NVR 共享录像存储目录，否则会报「打开视频失败，跳过」。

## systemd 常驻

参考 unit（按实际路径调整）：

```ini
# /etc/systemd/system/mibee-vision.service
[Unit]
Description=MiBeeVision Lite (video AI engine)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/opt/mibee-vision/mibee-vision-lite-v0.1.0-linux-aarch64 serve
Environment=ORT_DYLIB_PATH=/opt/mibee-vision/libonnxruntime-1.17.1-linux-aarch64.so
WorkingDirectory=/opt/mibee-vision
Restart=on-failure
RestartSec=5

# 建议按需加固：仅读录像目录 + 写自身数据目录
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/mibee-vision

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mibee-vision
journalctl -u mibee-vision -f     # 观察启动与录像段处理日志
```

## 端口

| 端口 | 用途 |
|---|---|
| `:9092` | 内嵌 Web 配置页（与 MiBeeNvr `:9090`、MiBeeSteward `:8080` 错开） |
