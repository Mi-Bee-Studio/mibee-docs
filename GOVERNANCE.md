# mibee-docs 治理规范（v1 · 2026-09-03）

> 本文件定义**分支模型、PR 合并与生产部署的责任边界**，适用于所有向本仓库提交文档的团队。
> 目录/内容规范见 [README](README.md)；各集合专属规范见 `mibeecam/CONVENTIONS.md`、`mibeelibs/CONVENTIONS.md`。

## 1. 角色与分支模型

```mermaid
flowchart LR
    subgraph teams [各文档团队（长期集成分支）]
        TE[team/espcam<br/>esp-cam 团队]
        TR[team/rpi-webui<br/>树莓派 + webui 团队]
        TL[team/libs<br/>库团队]
        TN[team/nvr<br/>NVR 团队]
    end
    PR[Pull Request<br/>docs(集合): 标题]
    MAIN[main（受保护）<br/>唯一生产源]
    PROD[官网生产<br/>www.mlsbs.top]
    TE --> PR
    TR --> PR
    TL --> PR
    PR -->|官网团队审查后 squash merge| MAIN
    MAIN -->|每小时自动同步<br/>（官网团队可手动触发）| PROD
```

| 分支 | 用途 | 谁可写 |
|------|------|--------|
| `main` | **唯一生产源**，受保护 | 仅经 PR 由**官网团队**合并；任何人不得直推 |
| `team/<团队名>` | 各团队的长期集成分支（如 `team/espcam`） | 本团队；从 `main` 创建并定期同步 |
| 特性分支 | 单个文档改动（可选），从团队分支拉出 | 任何人 |

## 2. 团队工作流（每个团队必须遵守）

1. **只在团队分支上更新**：日常改动提交到 `team/<团队名>`（新建团队从 `main` 拉出自己的分支并在此 issue 登记）。
2. 定期把 `main` 合入团队分支，保持可快进合并。
3. 改动就绪后**从团队分支向 `main` 提 PR**：标题 `docs(集合): 摘要`（如 `docs(mibeecam): …`、`docs(mibeelibs): …`）。
4. PR 描述自查并勾选：slug 前缀合规 / manifest 与文件一一对应 / mermaid 已在 [mermaid.live](https://mermaid.live) 校验 / 截图已脱敏 / 双语对应或已注明缺失侧 / 未触碰其他团队前缀。
5. 一个 PR 聚焦一个团队前缀；跨团队内容（如 `cam-` 公共页）的 PR 需在描述 @ 相关团队过目。
6. **等待官网团队审查合并**——团队不得自行合并；评论里的修改意见在团队分支上修复后 PR 自动更新。

## 3. 官网团队职责（合并守门人）

1. **合规审查**：按各集合 CONVENTIONS 与根 README 检查第 2.4 条自查项；红线（私有仓库信息、脱敏）一票否决。
2. **批准并 squash merge**；合并即视为放行上线。
3. **部署**：合并后生产在 **1 小时内**由服务器定时同步自动上线；需要立即上线时由官网团队手动触发同步（或按需走官网仓库发版）。
4. **紧急情况**：发现线上文档事故时，官网团队可直接操作（管理员通道），事后补 PR/issue 说明；常规回滚走 revert PR。
5. 治理规则本身的修改（分支模型、保护规则、审批人数）需开 issue 由官网团队裁定。

## 4. main 分支保护规则（已生效的 GitHub 设置）

- **禁止直推**：任何改动必须经 Pull Request（管理员保留紧急直推通道，见第 3.4 条）。
- **PR 必须获得 ≥1 个批准，且须包含 CODEOWNERS（官网团队）的批准**——即团队无法自行合并自己的 PR。
- **新推送自动失效既有批准**（dismiss stale reviews），改完需重新审查。
- **仅允许 squash merge**（历史线性、一 PR 一提交）；禁止 force push 与删除分支。

## 5. 团队分支登记

| 分支 | 团队 | 负责集合/前缀 |
|------|------|---------------|
| `team/espcam` | esp-cam 团队 | `mibeecam/` 的 `espcam-` `aicam-` `n16r8-` `luatos-` `seeed-` |
| `team/rpi-webui` | 树莓派 + webui 团队 | `mibeecam/` 的 `rpicam-` `webui-`（及后续新集合） |
| `team/libs` | 库团队 | `mibeelibs/` 全部前缀 |
| `team/nvr` | NVR 团队 | `mibeenvr/` 全部（含版本快照与 `versions.json`） |

新团队接入：从 `main` 创建 `team/<名称>` → 在本文件登记（PR 修改本表）→ 按 [README](README.md) 规范提交。
