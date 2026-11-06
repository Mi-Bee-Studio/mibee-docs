# 常见问题

### 和 NVR 内置的浏览器端 AI 检测是什么关系？

互补。浏览器端 AI 跑在浏览器 Web Worker 里，面向「打开页面时的实时画框」；MiBeeVision 是服务端独立进程，7×24 逐段分析录像并留存事件与快照，回写 NVR。两者独立开关互不影响。

### 需要 GPU 吗？

Lite 版不需要——CPU 推理（ONNX Runtime），普通 x86 服务器或 ARM 设备即可。GPU 加速（CUDA/TensorRT）与多摄像头在 Pro 版（规划中，发布时间待公布）。

### 支持 Windows / macOS 吗？

不提供。当前平台为 **Linux x86_64 / aarch64**。

### 配置页能打开，但永远不出事件？

按顺序检查两个前置：

1. `ORT_DYLIB_PATH` 是否指向随包的 `libonnxruntime-*.so`（缺失时推理库加载失败，管线处理不了录像段）；
2. 是否用了 `mbv_` 开头的 **API Key** 而不是 `mbs_` 网页登录 token（用错会 401，AI 状态永远为空）。

仍无事件时，确认传输方式与部署拓扑匹配：跨机部署必须用 `http_download`，`local_file` 需要与 NVR 共享录像目录。

### 回写 AI 事件返回 401？

用了 `mbs_` 开头的网页会话 token。`mbs_` 只对浏览器会话有效；请在 NVR「设置 → AI 检测 → 生成 API Key」生成 `mbv_` Key 给 Vision 使用。

### 怎么确认分析链路已经打通？

Vision 配置页状态区显示「管线 运行中 / NVR 已连接」，且几分钟后 NVR 录像列表的 AI 状态列出现 `completed`。

### 分析结果存在哪里？

- NVR 侧：录像列表 AI 标记 + AI 事件页；
- Vision 本地：结构化事件（SQLite/libSQL）+ 带框快照。

### 想分析已经录好的历史录像怎么办？

对存量录像的批量补分析在 Pro 版能力清单中（规划中，发布时间待公布）。当前 Lite 版面向新产生录像的持续分析。

### 商业使用有什么限制？

- 软件本体为闭源专有软件，版权归 MiBee Studio 所有；Lite 版随 MiBeeNvr 免费分发。
- 内置检测模型 YOLOv8n 采用 Ultralytics AGPL-3.0 许可，使用本程序进行检测即视为接受 AGPL-3.0 的相关义务；**商业闭源分发需另行取得授权**（或将模型替换为宽松许可模型）。

### 版本号怎么对应？

MiBeeVision 使用独立版本线 `mibee-vision-vX.Y.Z`，与 MiBeeNvr 的 `vX.Y.Z` 相互独立、各自演进。
