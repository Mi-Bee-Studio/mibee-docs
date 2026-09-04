# Rust 版（闭源发行）

MiBee Eye 除开源的 Go 版外，另有 Rust 实现的设备发行版。Rust 版以预编译固件 / 服务随机交付，**设备本体源码不公开**；其协议栈完全构建在本工作室开源的协议库之上，协议行为与开源库一一对应。

## 定位与开源边界

| 层 | 开放情况 | 说明 |
|----|---------|------|
| 设备服务本体（Rust） | 闭源发行 | 以预编译二进制交付，不提供源码构建 |
| 协议库（依赖） | **开源（MIT）** | 可独立集成到你自己的项目 |

Rust 版消费的两个开源库：

| 库 | crate | 作用 | 文档 |
|----|-------|------|------|
| [onvif-rs](https://github.com/mickeyzzc/onvif-rs) | `onvif-device-rs` | ONVIF Device 服务端：WS-Discovery、Media、PTZ、Imaging、Security | [ONVIF · Rust](https://www.mlsbs.top/docs/mibeelibs/onvif-rs-discovery) |
| [gb28181-rs](https://github.com/mickeyzzc/gb28181-rs) | `gb28181-rs` | GB28181 设备侧：注册 / 保活、目录、点播、回放 | [GB28181 · Rust](https://www.mlsbs.top/docs/mibeelibs/gb28181-rs-server) |

也就是说：设备在协议层面的互操作行为（线格式、状态机、应答语义）由这两个开源库定义并经 golden 测试约束——与设备源码是否公开无关。集成方可以直接依库开发、做兼容性验证，或为自己的产品复用同一套协议栈。库的总览见 [MiBee 库总览](https://www.mlsbs.top/docs/mibeelibs/libs-overview)。

## 能力概览

| 能力 | 说明 |
|------|------|
| ONVIF Profile S | Device / Media 服务 + WS-Discovery（UDP 组播与 HTTP 探测双路径） |
| GB28181 设备侧 | 注册与保活、目录与设备信息、实时点播、录像查询与回放 / 下载（UDP / TCP） |
| RTSP 直播 | H.264 / H.265 出流，支持 UDP 与 RTP over TCP |
| 本地录像 | 连续分段落盘 + 索引，作为国标回放 / 下载数据源，按保留天数与容量自动清理 |
| Web 管理 | 统一 Web API 规范 v1：会话认证、MSE 直播、配置读写、SSE 事件 |
| 快照 | JPEG 快照端点 |
| 移动侦测 | 帧间差异检测，本地运行 |
| 原生采集 | V4L2 / libcamera 原生绑定，无外部采集子进程 |

协议接入的操作指引与 Go 版共用：ONVIF 行为见 [ONVIF 合规性参考](rpicam-onvif-compliance.md)，GB28181 接入见 [GB28181 接入](rpicam-gb28181.md)（含 Rust 版 TOML 配置示例）。

## 与 Go 版对照

| 维度 | Go 版（开源） | Rust 版（闭源发行） |
|------|--------------|-------------------|
| 源码 | [mibee-eye-raspi-go](https://github.com/Mi-Bee-Studio/mibee-eye-raspi-go) 公开 | 不公开，预编译交付 |
| 相机采集 | 捆绑 mtxrpicam 子进程（自带 libcamera） | 原生 V4L2 / libcamera 绑定 |
| ONVIF 服务端 | onvif-go/v2 | onvif-rs（`onvif-device-rs`） |
| GB28181 设备侧 | gb28181-go | gb28181-rs |
| 配置格式 | YAML | TOML（键名与 Go 版一致） |
| Web 端口 | HTTP :8088 | HTTP :8088 |
| 配置生效 | 保存即自动重启 | 保存落盘后经 `POST /api/system/restart` 显式重启（重启后保持登录） |

两版共用同一份 [统一 Web API 规范](webui-spec.md)，前端为同一套嵌入实现——能力差异以各端 `/api/capabilities` 通告为准。

## 配置与使用

- 配置文件为 TOML；键名、语义与默认值和 Go 版一致（`camera` / `rtsp` / `onvif` / `device` / `logging` / `web` / `metrics` / `snapshot` / `gb28181` / `recording`）。键级参考直接阅读[配置文档](rpicam-configuration.md)与[配置参考](rpicam-config-reference.md)（示例为 YAML 语法，TOML 语法略异，`[section]` 键值对应）。
- 登录、配置、事件等端点契约见[统一 Web API 规范](webui-spec.md)；Rust 版保存配置后需显式重启生效。
- 设备与平台侧部署由交付渠道提供；如需评估或集成，联系我们获取发行包。

## 集成与定制

- 想在自有产品中复用同款协议行为：直接依赖上述两个开源库即可，无需依赖设备本体。
- 协议兼容性 / 互操作问题请到对应库的 GitHub issue 反馈；设备功能需求请联系 MiBee。
