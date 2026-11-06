# MiBeeNvr 协同

MiBeeVision 与 MiBeeNvr 是一对天然搭档：NVR 出画面（录像），Vision 出判断（检测/跟踪/规则），判断结果再回到 NVR 呈现。本章说明两者如何接线和结果流向。

## 接线三步

1. **在 NVR 生成 API Key**：NVR「设置 → AI 检测 → 生成 API Key」。生成的 Key 以 **`mbv_`** 开头。
2. **在 Vision 配置页填入**：浏览器打开 `http://<Vision主机IP>:9092`，填 NVR 地址、粘贴 `mbv_` API Key，点「测试连接并拉取摄像头」。
3. **选摄像头、画规则**：选择要分析的摄像头（Lite 为 1 路），配置规则名与区域顶点，点「保存」——管线自动重启并开始分析。

## ⚠️ 两个必读前置

漏掉任何一条，典型症状都是「配置页能打开，但永远不出事件」：

| 前置 | 错误做法的后果 |
|---|---|
| 必须设置 `ORT_DYLIB_PATH` 指向随包的 `libonnxruntime-*.so` | 推理库加载失败，管线无法处理录像段（配置页仍能打开） |
| 必须使用 `mbv_` 开头的 **API Key**，而不是网页登录 token（`mbs_` 开头） | 回写 AI 事件返回 401，NVR 录像的 AI 状态永远为空 |

`mbs_` token 只对浏览器网页会话有效；`mbv_` API Key 才是给 Vision 这类外部组件用的。

## 结果流向

```
NVR 录像段 ──► Vision 分析 ──┬──► 录像列表 AI 标记（处理中 / 完成）
                             ├──► AI 事件（区域入侵 / 越界 / 徘徊）
                             └──► 带框快照
```

- **录像列表**：每段录像出现 AI 状态标记，一眼看出哪些录像已经分析、发现了什么。
- **AI 事件**：在 NVR 的 AI 事件页查看规则触发的事件。
- **自检标准**：Vision 配置页状态区显示「管线 运行中 / NVR 已连接」，且几分钟后 NVR 录像列表的 AI 状态列出现 `completed`，即全链路打通。

## 部署拓扑

| 拓扑 | 传输方式（`[transport].type`） | 要求 |
|---|---|---|
| Vision 与 NVR 同机 | `local_file` | Vision 能直接读 NVR 录像存储目录（共享存储路径一致） |
| Vision 与 NVR 分机 | `http_download` | Vision 能通过网络访问 NVR；录像段先下载再分析 |

两种方式都在 Vision 配置页选择并保存，自动生效。跨机部署时若日志出现「打开视频失败，跳过： No such file or directory」，几乎都是误把 `local_file` 用在了没有共享存储的分机拓扑上。

## 版本搭配

MiBeeVision 使用独立版本线（`mibee-vision-vX.Y.Z`），与 NVR 版本号无关。请使用与 Vision Lite 同期或更新的 MiBeeNvr 版本，以确保 AI 状态与事件接口可用。
