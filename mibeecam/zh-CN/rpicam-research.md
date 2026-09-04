# 调研记录

> **历史决策记录（2026-05-29）。** 本页保存塑造 MiBee Eye 架构的开源调研与评估过程，反映调研当日各候选项目的状态。凡最终实现与当初选型有出入之处，均在文中就地标注。

## 概述

本文档记录 MiBee Eye 项目架构决策背后的开源调研与评估过程。调研聚焦 ONVIF 服务端库、树莓派相机方案、RTMP 推流能力，以及用自研 Go 实现整体替换 MediaMTX 的战略决策。

## 1. Go ONVIF 服务端库

### 候选库评估摘要

共评估多个 Go ONVIF 服务端库。核心要求：纯 Go 实现、ONVIF Profile S 合规、WS-Security 支持、适合资源受限的树莓派环境。

### 主要候选

| 库 | GitHub Stars | 许可证 | ONVIF 服务 | WS-Security | 备注 |
|---------|-------------|---------|----------------|-------------|------|
| `0x524a/onvif-go` | 380+ | MIT | Device、Media、PTZ、Imaging | ✅ | 纯 Go，功能全面但服务端模式是黑盒模拟器 |
| `ohcnetwork/mock-ptz-camera` | 180+ | MIT | Device、Media、PTZ、Imaging | ✅ | 完整虚拟 PTZ 相机，参考架构，可生产使用 |
| `github.com/halayun/onvif-server` | 45+ | Apache 2.0 | Device、Media、PTZ | ⚠️ | 文档有限，实现基础 |
| `github.com/simelo/rexx/onvif` | 12+ | MIT | Device、Media | ⚠️ | 维护极少，2021 年后未更新 |

### 功能对比

| 功能 | onvif-go | mock-ptz-camera | rexx/onvif | halayun/onvif-server |
|---------|----------|-----------------|------------|---------------------|
| **Device 服务** | ✅ 完整 | ✅ 完整 | ✅ 基础 | ✅ 完整 |
| **Media 服务** | ✅ 完整 | ✅ 完整 | ✅ 基础 | ✅ 完整 |
| **PTZ 服务** | ✅ 完整 | ✅ 完整（含数字 PTZ） | ❌ | ✅ 完整 |
| **Imaging 服务** | ✅ 完整 | ✅ 完整 | ❌ | ⚠️ 部分 |
| **WS-Discovery** | ✅ 内置 | ✅ 内置 | ❌ 手动 | ❌ 手动 |
| **WS-Security** | ✅ UsernameToken | ✅ UsernameToken | ❌ | ⚠️ 基础 |
| **GetStreamUri** | ✅ RTSP | ✅ RTSP | ✅ RTSP | ✅ RTSP |
| **GetProfiles** | ✅ 多 Profile | ✅ 多 Profile | ✅ 单 Profile | ✅ 多 Profile |
| **自动发现** | ✅ UDP Probe | ✅ UDP Probe | ❌ | ❌ |
| **SOAP Fault** | ✅ 完整 | ✅ 完整 | ❌ 基础 | ✅ 完整 |

### 结论

**选定**：`0x524a/onvif-go` 作为主服务端实现——ONVIF 覆盖全面、纯 Go 架构。

> **实际落地**：该库后被抽离为独立维护的分支 `mickeyzzc/onvif-go/v2`，产品当前实际消费的是它（见 [ONVIF · Go 库文档](https://www.mlsbs.top/docs/mibeelibs/onvif-go-architecture)）。

**参考**：`ohcnetwork/mock-ptz-camera` 用于 PTZ 实现模式与数字 PTZ 参考架构（PTZ 后来作为死代码移除）。

**落选**：`rexx/onvif`（范围窄、维护少）；`halayun/onvif-server`（WS-Security 不完整、文档缺口）。

## 2. 树莓派相机 ONVIF 方案

### 现有方案全景

树莓派生态已有若干相机 ONVIF 方案，从简单包装到完整实现都有。评估聚焦可维护性、ONVIF 合规度、相机集成方式与资源占用。

### 方案对比

| 方案 | 语言 | ONVIF 支持 | RTSP 来源 | 相机控制 | 维护 | 资源占用 |
|----------|----------|---------------|-------------|----------------|------------|----------------|
| **RPOS** | Python | 基础 Device | 外部 | 有限 | 中等 | 低（约 20MB） |
| **v4l2onvif** | Go | 仅 Device | V4L2 | 有限 | 活跃 | 中（约 30MB） |
| **MediaMTX** | Go | ❌ 无 ONVIF | 内置 | ⚠️ 仅 RTSP | 活跃 | 高（约 45MB） |
| **自研 Go** | Go | ✅ 完整 Profile S | 内置/外部 | ✅ 完整 | 活跃 | 目标约 30MB |
| **MotionEye** | Python | ❌ 无 ONVIF | FFmpeg | 有限 | 活跃 | 中（约 35MB） |
| **Zoneminder** | C++ | ❌ 无 ONVIF | FFmpeg | 丰富 | 活跃 | 高（约 100MB） |

### 集成方式分析

| 方案 | 相机接口 | 依赖 | 部署 | ONVIF 合规 |
|----------|------------------|--------------|-------------|----------------|
| **RPOS** | `raspistill` | Python、OpenCV | Python 运行时 | 仅 Device，部分 |
| **v4l2onvif** | V4L2 设备 | CGO、V4L2 | 静态二进制 | 仅 Device 服务 |
| **MediaMTX** | libcamera（C） | libcamera、rpicam | 系统服务 | 无（缺服务端模式） |
| **自研 Go** | mtxrpicam 子进程 | libcamera、rpicam | 静态二进制 | 完整 Profile S |
| **MotionEye** | `raspivid`/`rpicam` | FFmpeg、Python | Docker/系统服务 | 无（仅 RTSP 客户端） |

### 结论

**选定**：自研 Go 实现——对 ONVIF 合规与资源优化拥有完全控制权。

**关键洞察**：MediaMTX 缺少 ONVIF 服务端模式（issue #1402），尽管其 RTSP 推流能力优秀，仍必须替换。

## 3. RTMP 推流方案

### RTMP 推流需求

项目需要 RTMP 推流能力以对接云服务（阿里云、腾讯云等）。方案必须轻量、Go 原生，并能在资源受限硬件上与 ONVIF/RTSP 服务并存。

### RTMP 库评估

| 库 | GitHub Stars | 许可证 | 方式 | Go 原生 | 功能 | 维护 | 资源 |
|---------|-------------|---------|----------|-----------|----------|------------|---------|
| **MediaMTX pushTargets** | 11.3k+ | MIT | 内置 | ❌ Go/C++ | 基础推流鉴权、HLS | 活跃 | 约 5MB |
| **FFmpeg 子进程** | 20k+ | GPL | 外部 | ❌ C 二进制 | 通用、-c copy | 活跃 | 约 10MB RAM |
| **lal (q191201771/lal)** | 2.8k+ | MIT | Go 原生 | ✅ | RTMP/WebRTC/HLS、低延迟 | 活跃 | 约 15MB |
| **go2rtc (alexsnov/go2rtc)** | 1.2k+ | MIT | Go 原生 | ✅ | 多协议、WebRTC | 活跃 | 约 20MB |
| **livego (gwuhaolin/livego)** | 9.4k+ | MIT | Go/C++ | ⚠️ | RTMP/HLS/HTTP-FLV | 活跃 | 约 30MB |
| **github.com/aler9/gortspsrv-rtmp** | 80+ | MIT | Go 原生 | ✅ | RTSP 转 RTMP | 活跃 | 约 10MB |

### 方式对比

| 方案 | 架构 | 复杂度 | 流质量 | CPU 占用 | 内存占用 | 集成难度 |
|----------|--------------|-------------|-----------|--------------|-------------------|
| **MediaMTX pushTargets** | 集成 | 低 | 优秀 | 约 1-2% | 约 5MB | ✅ 极易 |
| **FFmpeg 子进程** | 外部 | 高 | 优秀 | 约 5-8% | 约 10MB | ⚠️ 进程管理 |
| **lal** | 集成 | 中 | 优秀 | 约 3-4% | 约 15MB | ✅ Go 原生 |
| **go2rtc** | 集成 | 中 | 良好 | 约 4-5% | 约 20MB | ✅ 多协议 |
| **livego** | 混合 | 高 | 优秀 | 约 6-8% | 约 30MB | ⚠️ C++ 复杂 |
| **gortspsrv-rtmp** | 集成 | 低 | 良好 | 约 2-3% | 约 10MB | ✅ 最小集成 |

### 结论

**选定**：`lal (q191201771/lal)` 作为主要 RTMP 推流库——Go 原生集成、资源占用适中、维护活跃。

> **实际落地**：最终发布实现是项目内自包含的 RTMP 推流客户端（`internal/rtmp`），未引入 lal 依赖；下述评估标准仍记录了为何倾向进程内 Go 方案。

**兜底**：FFmpeg 子进程作为通用兜底，用于需要特定转码选项的边缘场景。

## 4. MediaMTX 架构分析

### 组件分析

MediaMTX 是成熟的 Go 媒体服务器，当时在目标设备上承担相机采集与 RTSP 推流。架构分析聚焦：哪些组件可复用、哪些需要替换。

### 组件拆解

| 组件 | 可复用？ | 依赖 | 许可证 | 集成成本 |
|-----------|-----------|--------------|---------|-----------|
| **gortsplib v5** | ✅ 可 | 纯 Go | MIT | ✅ 直接集成 |
| **rpicamera 包** | ✅ 可 | libcamera C | MIT | ⚠️ 子进程适配 |
| **配置系统** | ❌ 否 | YAML、JSON | MIT | 换成自研配置 |
| **流管理** | ❌ 否 | 路径解析 | MIT | 换成自研管线 |
| **Web UI/API** | ❌ 否 | Go HTTP | MIT | 换成最小端点 |
| **WebRTC** | ❌ 否 | Go WebRTC | MIT | 本场景不需要 |

### rpicamera 包分析

`rpicamera` 包（C 编写）提供 MediaMTX 与 libcamera 之间的关键相机接口。要点：

```c
// 典型 rpicamera 使用模式
struct mtx_rpicam *rpicam = mtx_rpicam_new(width, height, fps);
mtxrpicam_set_option(rpicam, "brightness", "0");
mtxrpicam_set_option(rpicam, "contrast", "1.0");
mtxrpicam_start(rpicam);
// 帧数据经管道/UDP 获取
```

**优势**：

- 在树莓派 libcamera 集成上久经考验
- 子进程架构与 Go 兼容
- 基于管道的帧流协议
- MIT 许可允许复用

**局限**：

- 需要 C 交叉编译
- 接口固定（不支持动态切换相机）
- 单线程运行

### 复用策略

**复制/适配**：rpicamera 包（附署名）加 Go 包装
**复用**：gortsplib v5 承担 RTSP 服务功能
**替换**：配置、管理与 ONVIF 组件

### 架构决策

**为什么不直接用 MediaMTX**：关键阻塞点是 MediaMTX 缺少 ONVIF 服务端模式（issue #1402）。尽管 RTSP 推流优秀，它无法满足 ONVIF 设备发现与控制这一核心需求。

**复用什么**：久经考验的 rpicamera 子进程架构与基于管道的帧流
**替换什么**：除核心相机采集与 RTSP 推流外的全部组件

## 5. 相机采集方案

### 采集架构评估

为树莓派 3B + OV5647 评估了多种采集方式。评估标准：CGO 依赖、交叉编译复杂度、实际可靠性、可维护性。

### 采集方案对比

| 方式 | 语言 | 需要 CGO | 交叉编译 | 实际使用 | 可靠性 | 性能 | 复杂度 |
|--------|----------|---------------|------------------|------------------|-------------|------------|------------|
| **go4vl** | Go | ✅ 需要 | 复杂 | 较少 | 中 | 高 | 中 |
| **libcamera 直接** | Go/CGO | ✅ 需要 | 复杂 | 较少 | 中 | 高 | 高 |
| **MediaMTX rpicam** | C → Go | ❌ 不需要 | 简单 | 广泛 | 高 | 良好 | 低 |
| **FFmpeg raspivid** | C → Go | ❌ 不需要 | 简单 | 广泛 | 高 | 良好 | 中 |
| **rpicam-vid 子进程** | Go → C | ❌ 不需要 | 简单 | 广泛 | 高 | 良好 | 中 |
| **V4L2 /dev/video0** | Go | ✅ 需要 | 复杂 | 广泛 | 高 | 不定 | 中 |

### go4vl 分析

```go
// go4vl 示例 - 需要 CGO
import "github.com/vladimirvivien/go4vl/v4l2"

cap, err := v4l2.New("/dev/video0")
// 直接 V4L2 访问，零拷贝
```

**优点**：

- 直接内核接口，零拷贝帧
- 纯 Go API
- 性能好

**缺点**：

- 交叉编译需要 CGO
- 额外依赖：crossbuild-essential-arm64
- 与新内核的 libcamera 集成有限
- 错误处理复杂

### MediaMTX rpicam 集成

```bash
# 当时的 MediaMTX 方式
mtxrpicam --width 1280 --height 720 --fps 15 --pipe
# 帧数据经 stdout 管道获取
```

**优点**：

- 不需要 CGO
- libcamera 集成久经考验
- 子进程架构简单
- 资源占用低

**缺点**：

- 需要编译 C 二进制
- 子进程开销
- 相机接口固定

### 结论

**选定**：MediaMTX rpicam 子进程方式，理由：

- 无 CGO 依赖（纯 Go 交叉编译）
- 在树莓派 3B + libcamera 上可靠性经受过验证
- 与 Go 管线集成简单
- 内存占用低

**落选**：go4vl（CGO 复杂度）与 V4L2（与新 libcamera 内核不兼容）。

## 6. 架构决策记录

### 决策：整体替换 MediaMTX

**日期**：2026-05-29
**状态**：已批准
**影响**：高

### 决策背景

项目评估了两种主要路径：

1. **混合路径**：保留 MediaMTX 负责采集 + RTSP，外挂 Go ONVIF 代理
2. **完全替换**：用自研 Go 实现替换 MediaMTX

### 混合路径分析

**架构**：

```text
OV5647 相机 → MediaMTX rpicam → MediaMTX RTSP → Go ONVIF 代理 → NVR
```

**优点**：

- 风险较低（复用久经考验的组件）
- 可增量实施
- 现有 MediaMTX 配置可沿用

**缺点**：

- 内存占用更高（约 45MB + 15MB = 60MB）
- 进程复杂（两个服务）
- 相机协调困难
- 服务间时序问题

### 完全替换分析

**架构**：

```text
OV5647 相机 → 自研 rpicam → gortsplib v5 → Go ONVIF 服务 → NVR
```

**优点**：

- 单二进制部署（总计约 30MB）
- 相机控制完整集成
- 进程管理简化
- 总内存更低（约 30MB vs 约 60MB）
- 日志与监控统一
- 直接控制相机参数

**缺点**：

- 实现风险更高
- 需要维护 rpicam 子进程接口
- 失去 MediaMTX 久经考验的流管理

### 最终决策：完全替换

**选定**：完全替换路径

**理由**：

1. **资源优化**：单二进制把内存占用从约 60MB 降到约 30MB
2. **相机控制**：与 ONVIF 相机控制参数直接集成
3. **部署简化**：单个 systemd 服务，无需协调多服务
4. **性能**：直接基于管道的帧流消除进程间通信开销
5. **可维护性**：单代码库降低长期维护负担

**风险缓解**：

- 附署名复制久经考验的 rpicamera 包
- 用成熟的 gortsplib v5 承担 RTSP 功能
- 配套完整测试增量实现

**接受的取舍**：

- 失去 MediaMTX 久经考验的流管理
- 需要自研配置系统
- 子进程方式采集相机（相对直接内核访问）

### 实施优先级

1. **阶段 1**：基础 RTSP 服务器 + rpicam 集成
2. **阶段 2**：ONVIF 设备服务实现
3. **阶段 3**：相机控制（Imaging 服务）
4. **阶段 4**：RTMP 推流能力
5. **阶段 5**：进阶功能（快照）

### 成功标准

- 与 MiBee NVR 达成 ONVIF Profile S 合规
- RTSP 720p@15fps H.264 推流
- 内存占用 < 30MB
- 树莓派 3B 上 CPU < 25%
- 从 x86_64 交叉编译到 arm64

## 参考资料

1. [0x524a/onvif-go](https://github.com/0x524a/onvif-go) - 主 ONVIF 服务端库（后抽离为 `mickeyzzc/onvif-go/v2`）
2. [ohcnetwork/mock-ptz-camera](https://github.com/ohcnetwork/mock-ptz-camera) - Go ONVIF 参考架构（PTZ 模式未采用）
3. [bluenviron/mediamtx](https://github.com/bluenviron/mediamtx) - 当时的 RTSP 服务器（issue #1402）
4. [q191201771/lal](https://github.com/q191201771/lal) - 评估过的 RTMP 推流库（后被自研 `internal/rtmp` 客户端取代）
5. [vladimirvivien/go4vl](https://github.com/vladimirvivien/go4vl) - 评估过的 V4L2 采集（落选）
6. MiBee NVR - 定义 ONVIF 客户端需求的消费方

## 调研日期

本次调研完成于 2026-05-29，反映调研当日各候选库与方案的状态。调研过程包括库分析、代码审阅、架构评估与针对树莓派 3B 目标环境的资源规划。
