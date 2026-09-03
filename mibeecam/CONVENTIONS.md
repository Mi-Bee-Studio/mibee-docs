# mibeecam 摄像头总文档 · 底层规范（v1）

> 本文件是 `mibeecam/` 的**提交基线**：信息架构与排版可以大改，但下列规则不因重组而变化。
> 各仓库提交文档前必读；根目录 [README](../README.md) 定义全仓库通用规范，本文件在其上叠加 mibeecam 专属约束。

## 1. slug 前缀注册表（互斥，先到先得，不得占用他人前缀）

mibeecam 是**多合并项目容器**：每个"大集合"（一组相关相机项目）在命名空间上自成一体，
统一页用集合前缀，成员项目再用各自前缀。已注册：

| 前缀 | 归属 | 维护团队 |
|------|----------|----------|
| `espcam-` | ESP-Cam 大集合统一页（总览/架构/API/前端/知识库） | esp-cam 团队 |
| `cam-` | 总文档公共页（overview / nvr-integration 等） | 总文档维护者 |
| `aicam-` | ESP-Cam 集合成员：ai-thinker-esp32-cam | esp-cam 团队 |
| `n16r8-` | ESP-Cam 集合成员：esp32s3-n16r8-cam | esp-cam 团队 |
| `luatos-` | ESP-Cam 集合成员：luatos-esp32s3-a10-camera | esp-cam 团队 |
| `seeed-` | ESP-Cam 集合成员：seeed-esp32s3-cam | esp-cam 团队 |
| `rpicam-` | 树莓派子集（mibee-eye 系列；未来 mibee-rec 等树莓派系项目并入此前缀或注册并列前缀） | 树莓派 + webui 团队 |
| `webui-` | mibee-webui | 树莓派 + webui 团队 |

> 团队分工（2026-09-03 裁定）：esp-cam 团队维护 ESP-Cam 大集合（`espcam-` 统一页 + 四个 ESP32 相机分节）；树莓派 + webui 团队维护 `rpicam-`/`webui-`，并将在文档中心新建自己的集合项目。跨前缀的公共页（`cam-`）变更需两组都过目。

- slug 仅 `^[a-z0-9-]+$`：小写、数字、连字符；**不含**中文、下划线、斜杠。
- 文件名 = `slug.md`，与 manifest 的 `slug`/`file` 完全一致。
- **slug 稳定性**：已收录的 slug 不改名。确需变更属破坏性变更——PR 必须附新旧 slug 映射表，并同步改完所有站内链接；旧 slug 不得复用。
- 新集合接入：先注册集合前缀（开 issue 提案），manifest 里以独立 section 组呈现，组名含产品名。

## 2. 信息架构基线

侧边栏结构 = manifest 顺序。每个**大集合**遵循"统一 → 分板"模型：

```
总览（cam- 前缀公共页：总入口 / NVR 接入等）
├── 大集合（如 "ESP-Cam 系列"，espcam- 统一页）
│     统一页顺序：总览与板型矩阵 → 总架构 → 统一 API → 统一前端 → 知识库
├── 大集合成员分组（每组一个 section，组名含产品名，如 "ESP-Cam · Seeed XIAO ESP32-S3 Sense"）
│     组内推荐顺序：快速开始 → 用户手册 → 硬件 → 架构 → API → 知识库 → 故障排除
├── 其他大集合（如 "树莓派相机 rpi-cam"，并列 section）
└── 相机生态共享组件（webui- 等）
```

- 禁止把不同仓库的页面混入同一个 section；大集合统一页（espcam-）与成员页（aicam- 等）也分属不同 section。
- 内容只写**当前最新版本**：不保留旧版页面/旧版字段对照，历史差异收敛进统一页的"契约治理"小节。
- section `title` 用产品名，不带 emoji；item `title` 为简短名词（建议 ≤ 16 字），**不带仓库前缀**（前缀只体现在 slug）。

## 3. 页面骨架（每页必须满足）

1. **H1 唯一**且与 manifest `title` 一致；正文从 H2 起，层级最深到 **H4**。
2. H1 后 1–3 句导语：这页解决什么问题、适用型号。
3. 单页建议 ≤ 300 行（含代码）；超长必须拆页并在导语互相链接。
4. 操作步骤用有序列表；一步一动作；可执行命令给出完整命令块。
5. 代码块必须标注语言（```bash / ```c / ```json …）；mermaid 块标 ```mermaid。
6. 中文页用全角标点、中英文之间留空格；英文页用半角标点。

## 4. 图表规范

- **架构图、流程图、时序图、状态图：一律 mermaid 代码块，禁止位图/画板截图**。官网原生渲染，支持主题与放大。
- mermaid 节点文字保持简短（≤ 8 字/词）；图中不放长句，解释写在图外正文。
- 位图**仅限**真实 UI 截图与硬件照片：PNG/WebP，单图 ≤ 500 KB，宽边建议 1200px 级别。

## 5. 截图与脱敏（硬性红线，官网同步会整行删除违规文本但不会救截图）

截图入 `mibeecam/{lang}/images/`，正文相对引用 `images/xxx.png`；文件名 kebab-case 语义化（如 `web-dashboard.png`）。画面与文本中**不得出现**：

- 真实 IP/网段、MAC、序列号（替换为 `192.168.1.x` 等示例值）
- 真实设备名/房间名（替换为 前门/客厅 等通用名）
- WiFi SSID 与密码、token、密钥、二维码凭据
- 可识别家庭环境的画面（真实监控画面一律用演示数据或深色占位覆盖）

## 6. 链接

| 场景 | 写法 |
|------|------|
| 站内（mibeecam 内互链） | 相对文件名 `xxx.md` |
| 跨项目（如 mibeenvr 手册） | 完整 URL `https://www.mlsbs.top/docs/mibeenvr/{slug}` |
| 源码/仓库文件 | GitHub 绝对链接 |
| 页内锚点 | 标题锚（渲染器自动生成） |

## 7. 双语

- zh-CN 与 en-US 同 slug 一一对应为**目标态**；单语页允许临时存在，但只在该语言的 manifest 收录，且 PR 描述注明缺哪侧。
- 中英术语保持一致（同一概念全站同一词），新增术语在 PR 描述给出中英对照。

## 8. 版本快照与 latest 门控

沿用根 README：发版时固化 `vX.Y.Z/{zh-CN,en-US}/` 并登记 `versions.json`；**无版本目录的最新层跟随最新 release tag（tag 门控），不追 main**。

## 9. PR 流程

- 一切改动走 PR，标题 `docs(mibeecam): …`；单 PR 聚焦一个仓库前缀的变更（跨前缀重组拆多个 PR）。
- 大改（分组重组、slug 变更、新公共组）先开 issue 附目录提案，确认后再动手。
- 合并即上线（官网每小时同步）；mermaid 图提交前在 [mermaid.live](https://mermaid.live) 校验语法。
- 私有仓库信息红线名单见根 README；相机项目特别注意第 5 条截图红线。
