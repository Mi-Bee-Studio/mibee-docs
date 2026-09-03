# MiBee Docs — Mi&Bee Studio 官网文档中心

本仓库是官网（[mlsbs.top/docs](https://www.mlsbs.top/docs)）文档中心的**唯一文档源**。各项目手册按下方规范提交到本仓库固定位置，官网每小时自动同步（私有信息过滤 + 镜像），**文档更新不需要网站侧发版**。

## 目录规范

```
{projectId}/                      # 项目 ID（与官网一致：mibeenvr / mibeesteward / mibeehive）
├── versions.json                 # 版本索引（有多版本快照的项目必须有，见下）
├── zh-CN/                        # 最新版 · 中文
│   ├── manifest.json             # 目录清单（必需）
│   ├── {slug}.md                 # 文档页，文件名 = slug
│   └── images/                   # 截图等静态资源
├── en-US/                        # 最新版 · 英文（结构与 zh-CN 完全一致）
├── v0.12.0/                      # 历史版本快照（发版时固化，内部结构与最新版相同）
│   ├── zh-CN/ …
│   └── en-US/ …
└── v0.11.0/ …
```

- **最新版**直接放在语言目录下（无版本前缀）；**历史版本**每发一版固化一份 `vX.Y.Z/` 目录。
- 版本目录一旦固化**不再修改**（只读历史，勘误请在最新版修正）。

## manifest.json 格式

每个语言目录一份，官网侧边栏与路由都由它驱动：

```json
{
  "lang": "zh-CN",
  "title": "MiBeeNvr 用户手册",
  "sections": [
    {
      "title": "入门",
      "items": [
        { "slug": "intro", "file": "intro.md", "title": "产品介绍" }
      ]
    }
  ]
}
```

| 字段 | 说明 |
|---|---|
| `lang` | `zh-CN` 或 `en-US` |
| `title` | 手册名（侧边栏/标题用） |
| `sections[].title` | 分组标题 |
| `items[].slug` | URL 段（`/docs/{project}/{slug}`），同时是文件名（去掉 `.md`） |
| `items[].file` | 同目录下的文件名，与 slug 保持一致 |

## versions.json 格式

存在历史快照的项目必须维护（新版本在前）：

```json
{
  "project": "mibeenvr",
  "versions": [
    { "id": "v0.12.0" },
    { "id": "v0.11.0" }
  ]
}
```

官网文档页顶部的版本下拉即由此生成；无此文件的项目不显示版本切换。

## 提交规范（各项目仓库请遵守）

> **mibeecam（摄像头总文档）项目有额外底层规范**：slug 前缀注册表、信息架构基线、页面骨架、图表（mermaid 优先）与截图脱敏细则，见 [mibeecam/CONVENTIONS.md](mibeecam/CONVENTIONS.md) —— 向 mibeecam 提交前必读。


1. 文档提交到 `{projectId}/{zh-CN|en-US}/`，文件名与 manifest 的 `slug` 一致。
1. **同步以 release tag 为门（2026-09-03 裁定）**：各项目仓未打 release tag 之前，**不得**将该仓 main 分支的手册增量（页面刷新、新页、manifest/图片变更）同步到本仓 live 目录。live 目录必须始终等于该项目最近一个已发布 tag 的手册——官网"最新版"文档不得先于二进制描述未发布功能。增量在发版时随快照固化一起一次性同步（见下方发版流程）。
2. **站内互链**用相对文件名（如 `quickstart.md`、`../zh-CN/images/x.png`），官网渲染时自动解析；**站外链接**用完整 URL。
3. 图片放对应语言 `images/` 下，正文相对引用 `images/xxx.png`；截图需脱敏（IP、设备名、序列号等）。
4. 中英文档 slug 必须一一对应（英文缺页时官网自动回退中文并提示）。
5. **红线**：任何内容不得出现私有仓库信息（`MiBeeP2PServer`、`kite-agent`、`pcmonitor-luatos`、`demo-repository`）——官网同步时会**整行删除**命中行，请在源头杜绝。
6. **发版流程**：发布 vX.Y.Z 时，将当前 `{zh-CN,en-US}/` 原样固化为 `vX.Y.Z/{zh-CN,en-US}/`，并在 `versions.json` 数组头部登记 `{ "id": "vX.Y.Z" }`。

## 官网如何消费

- URL：`/docs/{project}/{slug}` 为最新版；`/docs/{project}/vX.Y.Z/{slug}` 为历史版本（页面右上侧边栏有版本下拉切换，历史版本页有提示横幅）。
- 同步机制：官网服务器每小时拉取本仓库 `main` 分支 tarball → 红线过滤 `.md` → 镜像到官网 `public/docs/`。提交后最迟 1 小时上线。
