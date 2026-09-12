# 故障排除

> 本页命令示例面向 **Go 版**服务（systemd 单元 `mibee-eye`）；Rust 版见 [Rust 版](rpicam-rs.md)，端口与协议排查方法通用。

mibee-eye（树莓派 ONVIF 相机服务）的常见问题和解决方案。相机 / 编码器 / RTSP / 快照 / HLS 类问题见[流媒体故障排除](rpicam-troubleshooting-media.md)。

## 快速健康检查

```bash
# 检查 mibee-eye 是否运行
systemctl status mibee-eye

# 检查相机设备
ls -la /dev/video0

# 检查网络连接
netstat -tlnp | grep -E '8554|8080|3702|8088'

# 检查内存使用
free -h

# 检查 CPU 使用
top -bn1 | head -20

# 检查 Web UI
curl -s -o /dev/null -w "%{http_code}" http://localhost:8088/
# 期望值：200（前端加载）

# 检查存活探针
curl -s http://localhost:8088/api/health

# 检查 HLS 流
curl -s http://localhost:8088/hls/stream.m3u8 | head -5

# 检查摄像头编码器是否正常工作（日志中应有 x264）
journalctl -u mibee-eye --since "1 minute ago" | grep -i "x264\|encoder\|h264"
```

## Web UI 登录问题

### 症状
- 显示登录页面但凭据无效
- 登录时显示 "Invalid credentials" 错误
- 无法访问 Web 管理面板

### 诊断
```bash
# 检查 Web UI 是否运行中
curl -s -o /dev/null -w "%{http_code}" http://localhost:8088/

# 检查认证端点
curl -s -X POST http://localhost:8088/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}'

# 检查配置文件中的凭据
grep -A2 'web:' config.yaml
```

### 解决方案
1. **默认凭据**：当 web.username/web.password 为空时，Web UI 回落使用 ONVIF 凭据。设置显式的 Web 凭据：
   ```yaml
   web:
     username: "admin"
     password: "your-password"
   ```
2. **会话过期**：登录基于服务端会话（cookie + CSRF，见[统一 Web API 规范](webui-spec.md)）。清除浏览器缓存并重新登录。
3. **检查配置**：确保 config.yaml 中 web.enabled: true
4. **重启服务**：sudo systemctl restart mibee-eye

## ONVIF 发现问题

### 症状
- NVR 无法发现相机
- WS-Discovery 探测失败
- 找不到 ONVIF 设备服务

### 诊断
```bash
# 检查 ONVIF HTTP 端口
netstat -tlnp | grep 8080

# 测试 UDP 多播端口
nc -ul 3702

# 检查 ONVIF 服务日志
journalctl -u mibee-eye -f

# 手动测试 ONVIF 端点
curl -X POST http://localhost:8080/onvif/device_service
```

### 解决方案
1. **网络问题**：检查多播路由
   ```bash
   # 如果需要，启用多播
   sudo sysctl -w net.ipv4.conf.all.mc_forwarding=1
   ```

2. **端口冲突**：更改 ONVIF 端口
   ```yaml
   onvif:
     port: 8081  # 如需要则更改
   ```

3. **防火墙阻塞**：允许 ONVIF 端口
   ```bash
   sudo ufw allow 8080/tcp
   sudo ufw allow 3702/udp
   ```

4. **发现超时**：如需要，在 NVR 配置中增加超时时间

## NVR 集成问题

### 症状
- NVR 显示相机但无法添加
- GetStreamUri 失败
- 身份验证问题

### 诊断
```bash
# 检查 config.yaml 中的 ONVIF 凭据
# 手动测试 ONVIF 客户端
curl -X POST -H "Content-Type: application/soap+xml" \
  -d "<soap:Envelope>...</soap:Envelope>" \
  http://localhost:8080/onvif/device_service

# 检查 RTSP URL 格式
echo "rtsp://localhost:8554/stream"

# 检查设备信息响应
curl -s http://localhost:8080/onvif/device_service | grep -i device
```

### 解决方案
1. **身份验证**：设置 ONVIF 凭据
   ```yaml
   onvif:
     username: "admin"
     password: "your-password"
   ```

2. **无效的 RTSP URL**：确保 URL 与配置匹配
   ```yaml
   rtsp:
     username: ""  # 如果没有身份验证则留空
     password: ""
   ```

3. **配置文件问题**：检查视频编码器配置
   ```yaml
   camera:
     width: 1280
     height: 720
     codec: "h264"
   ```

4. **设备信息**：更新设备元数据
   ```yaml
   device:
     name: "我的相机"
     manufacturer: "Raspberry Pi"
     model: "OV5647"
   ```

## GB28181 注册问题

### 症状
- 平台上设备不在线
- 注册后频繁掉线
- 点播无流

### 诊断
```bash
# 观察注册与保活日志
journalctl -u mibee-eye -f | grep -iE "gb28181|register|keepalive"

# 检查本地 SIP 端口
netstat -ulnp | grep 5060

# 检查状态页的 gb28181 字段（需登录）
curl -s http://localhost:8088/api/health
```

### 解决方案
1. **注册 401**：核对 `sip_domain`、`device_id`、`password` 与平台完全一致
2. **本地端口冲突**：平台与设备同机时修改 `local_sip_port`
3. **UDP 点播丢包**：改 `gb28181.transport: tcp`
4. **回放返回空**：未开启本地录像——录像与回放联动见 [GB28181 接入](rpicam-gb28181.md)

## 端口 9100 冲突

### 症状

- 指标端点无法启动
- 日志中出现 "Address already in use" 错误
- 服务启动但指标不可访问

### 诊断

```bash
# 检查端口 9100 是否被使用
netstat -tlnp | grep 9100

# 检查 prometheus-node-exporter
ps aux | grep node_exporter
```

### 根因

Prometheus node_exporter 默认使用端口 9100，与 mibee-eye 指标端点冲突。

### 解决方案

1. **禁用指标**：在 config.yaml 中设置 `metrics.enabled: false`

2. **更改指标端口**：使用不同的端口：
   ```yaml
   metrics:
     enabled: true
     port: 9101  # 或任何可用端口
   ```

## WiFi 稳定性问题

### 症状

- RTSP/HLS 流间歇性中断
- 客户端随机断开
- 负载下网络吞吐量下降

### 根因

RPi 3B WiFi 由于硬件限制（理论 270Mbps），在持续传输负载下会掉线。

### 解决方案

1. **尽可能使用以太网**：有线连接更稳定

2. **降低比特率/帧率**：针对 WiFi 降低相机设置：
   ```yaml
   camera:
     fps: 10
     bitrate: 1000000  # 1 Mbps
   ```

3. **重启服务以恢复**：如果 WiFi 掉线：
   ```bash
   sudo systemctl restart mibee-eye
   ```

## 性能与资源问题

### 症状
- 高内存使用
- 视频流延迟
- CPU 过载

### 诊断
```bash
# 检查内存使用
free -h

# 检查 CPU 使用
top -bn1 | grep -E 'mibee-eye|mediamtx'

# 检查 RTSP 连接
lsof -i :8554

# 检查网络带宽
ip -s link show wlan0
```

### 解决方案
1. **降低分辨率**：降低捕获设置
   ```yaml
   camera:
     width: 640
     height: 480
     fps: 10
     bitrate: 1000000  # 1 Mbps
   ```

2. **降低 RTSP 缓冲区大小**（高级）：在 config.yaml 中降低 subscriber_buffer_size
3. **监控资源**：添加监控
   ```bash
   # 每 5 秒监控内存
   watch -n 5 "free -h && ps aux | grep mibee-eye"
   ```

## 内存不足 (OOM) 问题

### 症状
- mibee-eye 进程被意外终止
- journalctl 日志中显示 "Killed"
- dmesg 显示 "Out of memory" 或 "oom-killer"
- 系统变得无响应

### 诊断
```bash
# 检查 OOM 终止事件
dmesg | grep -i "oom\|killed"

# 检查内存使用历史
free -h

# 检查内存消耗大的进程
ps aux --sort=-%mem | head -10
```

### 根因
树莓派 3B 只有 905MB RAM。如果另一个进程消耗过多内存（例如 prometheus-node-exporter-collectors 的 apt_info.py 使用 124MB），OOM 杀手会终止最大的进程。

### 解决方案
1. **检查 cron/周期性任务**：禁用不必要的定时器：
   ```bash
   systemctl list-timers | grep -E "apt|collect"
   sudo systemctl disable --now prometheus-node-exporter-apt.timer
   ```
2. **降低摄像头比特率**：在 config.yaml 中降低设置
3. **使用内存占用少的监控工具**：如果不需要，移除 prometheus-node-exporter-collectors
4. **添加交换空间**（最后手段）：512MB 交换文件提供 OOM 缓冲

## 调试模式和日志

### 启用调试日志
```bash
# 通过环境设置调试级别
MIBEE_EYE_LOGGING_LEVEL=debug ./mibee-eye

# 或者在 config.yaml 中
logging:
  level: "debug"
```

### 日志分析技巧
```bash
# 实时查看日志
journalctl -u mibee-eye -f

# 过滤错误消息
journalctl -u mibee-eye | grep ERROR

# 查找超时模式
journalctl -u mibee-eye | grep -i timeout

# 检查资源警告
journalctl -u mibee-eye | grep -i "memory\|cpu"
```

## 常见错误消息

### 相机访问问题
```text
ERROR: camera device not available
```
- 解决方案：停止 MediaMTX，检查设备路径

### 端口已使用
```text
ERROR: address already in use
```
- 解决方案：在配置中更改端口或停止冲突服务

### 网络问题
```text
ERROR: connection refused
```
- 解决方案：检查防火墙，网络连接

### 内存问题
```text
WARNING: high memory usage detected
```
- 解决方案：降低分辨率、FPS 或比特率

### 编码器创建错误
```text
camera: mtxrpicam error: encoder_create(): unable to activate output stream
```
- 原因：捆绑的 libcamera 共享库缺失或 LD_LIBRARY_PATH 未设置
- 解决方案：参见[流媒体故障排除](rpicam-troubleshooting-media.md)的「摄像头编码器问题」

### 共享库未找到
```text
error while loading shared libraries: libcamera.so.9.9: cannot open shared object file
```
- 原因：LD_LIBRARY_PATH 未包含 deploy/bin/ 目录
- 解决方案：在 systemd 服务中设置 `Environment=LD_LIBRARY_PATH=<deploy-path>/bin`

### HLS 错误
```text
WARNING: HLS bridge not started
```
- 解决方案：确保 `hls.enabled: true` 且 RTSP 服务器正在运行以向 AUHub 订阅者提供数据

## 系统状态命令

```bash
# 完整系统健康检查
echo "=== 系统状态 ==="
echo "相机设备："
ls -la /dev/video0 2>/dev/null || echo "未找到相机"

echo "服务："
systemctl is-active mibee-eye mediamtx

echo "网络："
netstat -tlnp | grep -E '8554|8080|3702'

echo "Web UI:"
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8088/

echo "HLS 状态:"
curl -s http://localhost:8088/hls/stream.m3u8 > /dev/null 2>&1 && echo "HLS 活动" || echo "HLS 未活动"

echo "内存："
free -h

echo "相机进程："
pgrep -f mibee-eye || echo "MiBee Eye 未运行"

echo "冲突进程："
lsof /dev/video0 2>/dev/null | grep -v mibee-eye || echo "无冲突"

echo "编码器状态："
journalctl -u mibee-eye --since "5 minutes ago" | grep -i "x264\|encoder" | tail -3

echo "库解析："
LD_LIBRARY_PATH=~/mibee-eye/deploy/bin ldd ~/mibee-eye/deploy/bin/mtxrpicam 2>&1 | grep -E "found|libcamera"
```

## 联系支持

如果问题持续存在：
1. 检查日志：`journalctl -u mibee-eye`
2. 包含系统信息：`uname -a`，`dpkg -l | grep mibee-eye`
3. 提供确切的错误消息和重现步骤
4. 包含配置文件（删除敏感数据）
