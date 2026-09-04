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
4. PR 描述自查并勾选：slug 前缀合规 / manifest 与文件一一对应 / mermaid 已在 [mermaid.live](https://mermaid.live) 校验 / 截图已脱敏 / 双语对应或已注明缺失侧 / 未触碰其他团队前缀 / **泄密扫描（CI `security-check`）绿色**。
5. 一个 PR 聚焦一个团队前缀；跨团队内容（如 `cam-` 公共页）的 PR 需在描述 @ 相关团队过目。
6. **等待官网团队审查合并**——团队不得自行合并；评论里的修改意见在团队分支上修复后 PR 自动更新。

## 3. 官网团队职责（合并守门人）

1. **合规审查**：按各集合 CONVENTIONS 与根 README 检查第 2.4 条自查项；红线（私有仓库信息、脱敏、泄密）一票否决。
2. **批准并 squash merge**；合并即视为放行上线。**合并前必须完成 §3.6 安全审核**。
3. **部署**：合并后生产在 **1 小时内**由服务器定时同步自动上线；需要立即上线时由官网团队手动触发同步（或按需走官网仓库发版）。
4. **紧急情况**：发现线上文档事故时，官网团队可直接操作（管理员通道），事后补 PR/issue 说明；常规回滚走 revert PR。
5. 治理规则本身的修改（分支模型、保护规则、审批人数）需开 issue 由官网团队裁定。

### 3.6 合并前安全审核（2026-09-05 起强制）

**每个文档 PR 合并前必须通过泄密审核**，含自动与人工两道关：

**① 自动门禁（CI `security-check`，必须绿色）**：`python3 scripts/security-check.py`
扫描全仓库，命中以下任一类即失败：

- 真实内网网段（`192.168.62/63.x` —— 本项目公开脱敏规范认定的真实家庭网段，文档一律写 `192.168.1.x`）；
- 已知基础设施地址（生产服务器等，见脚本 `FORBIDDEN_HOSTS`）；
- 已知泄露标识（真实 SSID/设备名等，见脚本 `FORBIDDEN_STRINGS`）；
- 密钥/token 格式（GitHub/AWS/OpenAI/Slack/Google/GitLab/npm/DO、JWT、PEM 私钥、SSH 密钥）；
- 非文档保留段的公网 IPv4（文档示例 IP 只允许 `192.0.2.x` / `198.51.100.x` / `203.0.113.x`）；
- **凭据取值白名单外的新密码**：文档中出现的 password/密码/secret/token 取值，只允许
  (a) 三个**官方公开默认密码**：`mibeecam2026` / `mibeestudio2026` / `mibeehome2026`（须注明“首次配置后立即修改”）；
  (b) 第三方厂商公开默认值（如 `admin`/`123456`，仅限摄像头接入等事实性文档）；
  (c) 明确占位符（`your-*`、`<...>`、`changeme` 等）。
  确需新增取值：在脚本白名单登记并在 PR 描述说明理由，由守门人审查放行。

**② 人工审查（守门人逐项过目，模式扫描覆盖不了的）**：真实姓名/住址/手机号/邮箱、
真实 SSID 与设备名、设备序列号/SN、真实 MAC 地址、内网拓扑描述（主机名、端口清单、服务分布）、
云服务器/域名+凭据组合、截图是否走过脱敏管线（参考官网仓库 `tmp/docshots` 流程）。

**免责例外**：仅三个官方公开默认密码允许出现在公开文档；其余任何真实凭据——
无论看起来多“内部”——一律不得合并。误报通过白名单登记解决，不允许用“跳过扫描”放行。

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
| `team/steward` | steward 团队 | `mibeesteward/` 全部 |
| `team/hive` | hive 团队 | `mibeehive/` 全部 |

新团队接入：从 `main` 创建 `team/<名称>` → 在本文件登记（PR 修改本表）→ 按 [README](README.md) 规范提交。
