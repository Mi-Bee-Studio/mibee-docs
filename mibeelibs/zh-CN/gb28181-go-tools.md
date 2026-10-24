# Release 工具包：device-register / platform-uas / psmux-rtp

`examples/` 里的三个可执行示例同时是**互操作工具**——每次
[release](https://github.com/mickeyzzc/gb28181-go/releases) 都以七平台预编译包
发布（`gb28181-go-tools_<os>_<arch>.tar.gz`，Windows 为 `.zip`，附
`SHA256SUMS`）：linux amd64/arm64/arm、darwin amd64/arm64、windows
amd64/arm64。`CGO_ENABLED=0` 纯静态零依赖——下载解包即用，不需要 Go
工具链，拷到树莓派、NAS、测试笔电上都行。

## 哪个工具干哪件事？

| 你想…… | 运行 |
|---|---|
| 在没有真摄像头的情况下验证一个 GB/T 28181 平台的设备侧行为 | `device-register` |
| 给设备端 / 级联端一个可注册、可拉流的最小平台 | `platform-uas` |
| 快速验证 PS 复用 + RTP 打包的媒体路径（与设备直播推流同一条代码路径） | `psmux-rtp` |
| 看三方怎么集成（每个工具就是一份能跑的最小集成样例） | 三个源码都是 |

与 [onvif-go CLI 工具](onvif-go-cli.md) 的分工：那边是**运维/诊断**工具
（发现、体检、诊断、虚拟相机），这边是**协议角色模拟**工具——两个库各自
面向自己的协议栈。

## 安装

从 release 取预编译包：

```bash
gh release download v0.13.0 -R mickeyzzc/gb28181-go \
  -p 'gb28181-go-tools_linux_amd64.tar.gz'
tar xzf gb28181-go-tools_linux_arm64.tar.gz   # 三个二进制都在包里
sha256sum -c SHA256SUMS                        # 校验
```

或按版本锚定从源码装（examples 也是 module 内的包）：

```bash
go install github.com/mickeyzzc/gb28181-go/examples/device-register@v0.13.0
```

或检出后直接 `go run ./examples/<tool>`。

## device-register —— 设备端（UAC）模拟器

一台"虚拟摄像头"：向平台 REGISTER（digest 认证）→ 周期保活 →
应答 Catalog / DeviceInfo 查询 → 平台 INVITE 后按 **1 IDR + 1 P 帧/秒**
的节奏推 RTP/PS 流（合成帧，无需采集设备）。

配置在 `main.go` 的 `cfg` 里改（当前不带命令行旗标）：
平台地址与端口、设备/通道 20 位 ID、`Password`（与平台约定的摘要口令，
**占位值必须改**）、`Transport`（`udp` | `tcp` | `tls`——SIPS，
GB/T 28181-2022）。

典型用途：平台侧联调与回归（目录分页、INVITE、TCP/SIPS 传输）；
也是 `device` 包的最小集成参考——把合成帧循环换成你的编码器输出、
接到 `device.FrameHub` 就是生产接线。

## platform-uas —— 平台端（UAS）最小服务器

设备注册进来的那半边：SIP 监听 `:5060`（`ServerID 34020000002000000001`、
realm `3402000000`、digest 口令同样要改占位值）、保活存活窗、目录处理、
INVITE 起流进 FrameHub 扇出、媒体端口 `30000-30050`（UDP）、
告警经 `EventBus` 订阅打印。设备/通道持久化是 `DeviceStore` 接缝——
示例给的是内存实现，真实宿主在这里接数据库。

典型用途：让 `device-register`、级联客户端或你的设备实现有一个
"对端"可打；验证设备侧行为时不用真实平台。

## psmux-rtp —— 媒体路径自检

完全自包含的回环：把合成 H.264 access unit 复用成 MPEG-PS、
RTP 打包（PT 96）发给进程内的 UDP 接收端，每 50 包打印
包数/字节数统计。不监听端口、不依赖对端——一条命令证明
`psmux` 复用与 RTP 打包路径在当前平台工作正常（设备角色的
直播推流、级联角色的流转发用的都是这条路径）。

典型用途：新架构/新平台上的二进制冒烟；媒体面问题的最小复现载体。

## 共同说明

- 三个工具都**先读源码再用**：它们首先是 examples（集成样例），
  其次才是工具；改动配置即改即用（`go run`）。
- 口令占位值（`your-platform-password`）不改无法对接成功。
- 工具随 release 演进，能力面跟随库版本——各版本差异见仓库
  [CHANGELOG](https://github.com/mickeyzzc/gb28181-go/blob/main/CHANGELOG.md)。
