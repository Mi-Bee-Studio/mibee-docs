# N16R8 Web UI

N16R8 使用家族统一 SPA（三 S3 仓共享的四文件前端，见 [统一前端设计](espcam-webui.md)）。本页讲这套 UI 在本板上的呈现：哪些页签出现、哪些控件因能力位/字段缺省隐藏、以及 AI 交互。

## 页签构成（能力驱动）

| 页签 | 本板 | 依据 |
|---|---|---|
| 预览（MJPEG + AI 叠加） | ✅ | 恒有 |
| AI 检测 | ✅ | `capabilities.ai == true` |
| 相机设置 | ✅ | 恒有（分辨率表动态拉取） |
| 系统设置（WiFi/密码/RTSP/ONVIF） | ✅ | 恒有 |
| 存储 | ❌ 隐藏 | `capabilities.sd == false` |
| 录像 | ❌ 隐藏 | `capabilities.recording` 缺省 |
| 音频 | ❌ 隐藏 | `capabilities.audio == false` |

同一套四文件在 seeed 上会多出存储/录像/音频页签——差异全部来自运行时能力探测，**不是两套代码**。

## 预览页的 AI 叠加

MPEG 图像之上叠一块画布，`GET /api/ai/status` 每 500 ms 轮询（仅 AI 开启时）：

- 人脸：绿框 + 置信度
- 移动：分数条（0-100）
- 二维码：解码文本

`seq` 递增号用于丢弃迟到帧，避免叠加层与视频错位。

## 相机设置的 AI 联动

分辨率下拉框来自 `supported_resolutions`（本板 0-15 档）。任一 AI 特性开启时：

- 非 VGA 档位在下拉框中禁用（后端同样 400 兜底）
- 反向联动：把分辨率调到非 VGA 会提示先关 AI

传感器微调滑杆（亮度/对比度/饱和度/锐度，-2…+2）、镜像/翻转开关即时生效，不需要重配。

## 系统设置页的板级字段

- RTSP 凭证（`rtsp_user`/`rtsp_pass`，独立于 Web 管理密码）
- ONVIF 开关
- AI 三个特性的总开关
- 修改密码模态（家族统一流程：旧密码 + 新密码 + 确认）

WiFi 区块只显示单网络表单——本板无双 WiFi 字段（`wifi_ssid_2` 不在 `GET /api/config` 响应里，控件随之缺省）。

## 鉴权与会话

密码存 `sessionStorage`，写操作自动附 `X-Password`；401 引导到系统页。详见[统一前端设计](espcam-webui.md)。

## 界面

![主面板](images/index-dashboard.png)

![预览页](images/preview-page.png)

![配置页](images/config-page.png)

相关阅读：[Web API](n16r8-web-api.md) · [统一前端设计](espcam-webui.md)
