# mibeelibs 库总文档 · 底层规范（v1）

> 本文件是 `mibeelibs/` 的**提交基线**，由库团队独立维护；通用规范见根目录 [README](../README.md)，摄像头集合的平行规范见 [mibeecam/CONVENTIONS.md](../mibeecam/CONVENTIONS.md)。

## 1. slug 前缀注册表（互斥，先到先得）

| 前缀 | 归属仓库 | 说明 |
|------|----------|------|
| `libs-` | 集合公共页（overview / 选型等，由库团队管理） | 集合门面 |
| `gb28181-go-` | mickeyzzc/gb28181-go | GB28181 Go 实现 |
| `gb28181-rs-` | mickeyzzc/gb28181-rs | GB28181 Rust 设备侧 |
| `onvif-go-` | mickeyzzc/onvif-go | ONVIF Go 实现 |
| `onvif-rs-` | mickeyzzc/onvif-rs | ONVIF Rust Device 服务端 |
| `fingerprints-` | Mi-Bee-Studio/mibee-fingerprints-go | 指纹规则引擎 |

- 新库接入：先开 issue 注册前缀（模式与 mibeecam 一致），manifest 中以独立 section 呈现；本集合定位是**从产品抽离的库**，独立工具型仓库（exporter/网关等）不在此集合，需要时另立集合。
- slug 仅 `^[a-z0-9-]+$`；文件名 = `slug.md` = manifest `slug`/`file`。
- **slug 稳定性**：已收录 slug 不改名；确需变更属破坏性变更，PR 附新旧映射表并改完全部内链。

## 2. 信息架构基线

```
总览（libs-overview：生态图 + 选型建议）
└── 每库一个 section（组名 = "协议 · 语言"，如 "GB28181 · Go"）
      组内推荐顺序：快速开始/总览 → 架构 → 协议与核心概念 → 功能模块 → 用法/API → 测试 → 故障排除
```

- 库文档的核心读者是**集成方开发者**：每页先回答"这个东西解决什么问题、我怎么用"，再展开 internals。
- 内容只写**当前最新发布版本**；版本间差异放 CHANGELOG/发版说明，不留在正页。

## 3. 页面骨架

1. H1 唯一且与 manifest `title` 一致；正文 H2 起，最深 H4。
2. H1 后 1–3 句导语：解决什么问题、适用场景、成熟度状态。
3. 单页 ≤ 300 行；超长拆页。
4. **代码示例硬性要求**：可编译、import 路径与最新发布 tag 一致；Go 给 `go get` 版本锚定的安装行，Rust 给 `cargo add`；示例不依赖未导出符号。
5. 代码块必标语言；中文页全角标点 + 中英文间空格，英文页半角。

## 4. 图表规范（库文档的重头）

- **协议交互一律 mermaid `sequenceDiagram`**（注册/保活/INVITE 点播/回放控制、WS-Discovery/GetCapabilities/GetProfiles 握手等）——这是库文档最重要的图，逐条消息标注方向与关键头字段。
- 架构/模块关系用 `flowchart`；状态机（会话状态、重连策略）用 `stateDiagram-v2`。
- 禁止用位图画协议图/架构图；位图仅限真实抓包截图、UI 截图等确有必要的内容。
- 抓包截图必须脱敏（见下条）。

## 5. 脱敏红线

- 示例配置、日志、抓包、命令输出中不得出现：真实 IP/网段、设备序列号、SIP ID/国标编码（用 `3402000000...` 示例段）、证书/密钥、token。
- 私有仓库信息红线名单见根 README（同步时整行删除）。

## 6. 链接

| 场景 | 写法 |
|------|------|
| 集合内互链 | 相对文件名 `xxx.md` |
| 跨集合（mibeecam / mibeenvr 等） | 完整 URL `https://www.mlsbs.top/docs/{project}/{slug}` |
| 上游仓库/源码/ISSUE | GitHub 绝对链接 |

## 7. 双语与版本

- zh-CN / en-US 同 slug 一一对应为目标态；单语页临时允许但须在 manifest 只收录存在语言并在 PR 注明。
- 库发版打 tag 时按根 README 固化 `vX.Y.Z/` 快照并登记 `versions.json`（latest 跟最新 release tag）。
- **文档与 tag 同步**：API/行为描述不得先于未发布代码；发版 PR 内同步更新文档并固化快照。

## 8. PR 流程

- 标题 `docs(mibeelibs): …`；单 PR 聚焦一个库前缀。
- 新页面/新前缀/分组重组先开 issue 提案。
- mermaid 提交前在 [mermaid.live](https://mermaid.live) 校验；合并后官网一小时内自动上线。
