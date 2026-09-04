# 流媒体故障排除

MiBee Eye 相机检测、编码器、RTSP、快照与 HLS 相关问题的症状、诊断与解决方案。Web UI / ONVIF / NVR / 系统资源类问题见[故障排除](rpicam-troubleshooting.md)；GB28181 相关见 [GB28181 接入](rpicam-gb28181.md)。

## 相机检测问题

### 症状
- NVR 中找不到相机
- MiBee Eye 日志显示 "camera not detected"
- 流显示黑屏

### 诊断
```bash
# 检查相机设备是否存在
ls -la /dev/video0
# 应显示 /dev/video0 字符设备

# 直接使用 libcamera 测试相机
rpicam-hello

# 检查设备树覆盖层
cat /boot/firmware/config.txt | grep dtoverlay
# 应显示：dtoverlay=ov5647

# 检查内核相机支持（如果设备通过 libcamera 工作，可忽略）
vcgencmd get_camera
```

### 解决方案
1. **MediaMTX 冲突**：首先停止 MediaMTX
   ```bash
   sudo systemctl stop mediamtx
   sudo systemctl disable mediamtx
   ```

2. **缺少 DT 覆盖层**：添加到 config.txt
   ```bash
   sudo nano /boot/firmware/config.txt
   # 添加：dtoverlay=ov5647
   sudo reboot
   ```

3. **相机模块未连接**：检查 CSI 电缆

4. **设备路径错误**：更新 config.yaml
   ```yaml
   camera:
     device: "/dev/video0"  # 或你的相机设备
   ```

## 摄像头编码器问题

### 症状
- 服务启动但日志显示 "encoder_create(): unable to activate output stream"
- RTSP 流连接但无视频数据
- 快照端点返回 503 Service Unavailable

### 诊断
```bash
# 检查 mtxrpicam 是否能找到 libcamera
LD_LIBRARY_PATH=/home/pi/mibee-eye/deploy/bin ldd ~/mibee-eye/deploy/bin/mtxrpicam
# 如果显示 "libcamera.so.9.9 => not found"，说明捆绑库缺失

# 检查捆绑的 libcamera 文件是否存在
ls -la ~/mibee-eye/deploy/bin/libcamera*.so*

# 检查 systemd 服务中的 LD_LIBRARY_PATH
grep LD_LIBRARY_PATH /etc/systemd/system/mibee-eye.service

# 直接使用 rpicam-vid 测试摄像头
rpicam-vid -t 1000 --width 1280 --height 720 -o /dev/null

# 检查系统 libcamera 版本（可能与捆绑版本不同）
dpkg -l | grep libcamera
```

### 根因
mtxrpicam 二进制文件动态链接的是 `libcamera.so.9.9`，这与系统安装的 libcamera（Debian 13 提供的是 `libcamera.so.0.7`）不同。捆绑版本必须存在于 `deploy/bin/` 目录中，且 `LD_LIBRARY_PATH` 必须指向该目录。

### 解决方案
1. **缺少捆绑库**：从 mediamtx-rpicamera 发布版重新部署
   ```bash
   # 在工作站上下载并解压
   gh release download v2.6.0 --repo bluenviron/mediamtx-rpicamera \
     --pattern "mtxrpicam_64.tar.gz"
   tar xzf mtxrpicam_64.tar.gz

   # 复制捆绑库到设备
   scp mtxrpicam_64/libcamera*.so* <your-rpi-user>@<your-rpi-ip>:~/mibee-eye/deploy/bin/
   scp mtxrpicam_64/mtxrpicam <your-rpi-user>@<your-rpi-ip>:~/mibee-eye/deploy/bin/

   # 重启服务
   sudo systemctl restart mibee-eye
   ```

2. **LD_LIBRARY_PATH 未设置**：验证 systemd 服务配置
   ```bash
   # 应包含：Environment=LD_LIBRARY_PATH=/path/to/deploy/bin
   systemctl cat mibee-eye

   # 如果缺失，编辑服务文件
   sudo systemctl edit mibee-eye --force
   # 添加：Environment=LD_LIBRARY_PATH=/home/pi/mibee-eye/deploy/bin
   sudo systemctl daemon-reload
   sudo systemctl restart mibee-eye
   ```

3. **摄像头被其他进程占用**：停止 MediaMTX
   ```bash
   sudo systemctl stop mediamtx
   sudo systemctl disable mediamtx
   # 验证摄像头已释放
   lsof /dev/video0
   ```

## RTSP 流媒体问题

### 症状
- RTSP 流无法访问
- 连接超时
- NVR 无法连接到流

### 诊断
```bash
# 检查 RTSP 端口是否在监听
netstat -tlnp | grep 8554

# 本地测试 RTSP 连接
ffplay rtsp://localhost:8554/stream

# 检查防火墙规则
sudo ufw status

# 检查相机独占性
lsof /dev/video0
```

### 解决方案
1. **端口冲突**：在 config.yaml 中更改 RTSP 端口
   ```yaml
   rtsp:
     port: 8555  # 如需要则更改
   ```

2. **防火墙阻塞**：允许 RTSP 端口
   ```bash
   sudo ufw allow 8554/tcp
   ```

3. **相机访问冲突**：确保只有一个进程使用 /dev/video0
   ```bash
   sudo systemctl stop mediamtx  # 如果正在运行
   ```

4. **网络问题**：从客户端系统检查
   ```bash
   telnet <camera-ip> 8554
   ```

## 快照问题

### 症状
- 快照端点返回错误
- NVR 无法捕获图像

快照端点挂在 Web 端口 :8088（遗留 `/snapshot` 无认证端点 + `/api/cameras/0/snapshot` 认证端点，见[统一 Web API 规范](webui-spec.md)）。

### 诊断
```bash
# 检查快照端点
curl -I http://localhost:8088/snapshot

# 测试快照端点并检查响应
curl -s -w "\nHTTP 状态: %{http_code}\n" http://localhost:8088/snapshot -o /dev/null
# HTTP 200 + "image/jpeg" = 正常工作
# HTTP 503 = 摄像头未提供帧（检查编码器）
```

### 解决方案
1. **相机未运行**：确保 MiBee Eye 处于活动状态
   ```bash
   sudo systemctl restart mibee-eye
   ```

2. **分辨率问题**：调整相机分辨率
   ```yaml
   camera:
     width: 1280
     height: 720
   ```

3. **编码器未产生帧**：检查日志并验证 mtxrpicam 正常工作
   ```bash
   journalctl -u mibee-eye --since "5 minutes ago" | grep -i "encoder\|h264"
   ```

## HLS 实时预览问题

### 症状
- Web UI 显示黑色视频播放器
- Web UI 中显示 "HLS not available" 消息
- 浏览器控制台显示 hls.js 错误

### 诊断
```bash
# 检查 HLS HTTP 端点
curl -s http://localhost:8088/hls/stream.m3u8

# 验证配置中启用了 HLS
grep -A2 'hls:' config.yaml

# 检查 MiBee Eye 日志中的 HLS 错误
journalctl -u mibee-eye --grep "HLS"

# 检查 RTSP 流是否在为 HLS 提供数据（必须运行）
journalctl -u mibee-eye --grep "AUHub"
```

### 解决方案
1. **HLS 未启用**：在 config.yaml 中启用 HLS
   ```yaml
   hls:
     enabled: true
     segment_duration: 2s
   ```

2. **RTSP 源不可用**：首先确保 RTSP 流正常工作：
   ```bash
   ffprobe rtsp://localhost:8554/stream
   ```

3. **浏览器兼容性**：HLS 使用纯 Go MPEG-TS 分段器通过 HTTP 提供。确保浏览器支持 MSE 或使用 hls.js 库。

4. **重启 MiBee Eye**：sudo systemctl restart mibee-eye

5. **调试日志**：设置 `logging.level: debug` 以查看分段器错误
