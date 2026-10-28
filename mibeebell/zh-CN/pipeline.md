# 告警管道（四道闸 + L0）

管道是 MiBeeBell 的核心差异化：**ntfy / Gotify 类工具"来一条推一条"，MiBeeBell 在通知出口前放了一条可配置的管道**——去抖、聚合、冷却、升级、静默，加上一层永远在线的纯算法降噪（L0）。设计原则：全部纯函数 + 显式状态，每道闸可表驱动测试；参数全部规则级可覆盖。

## 处理顺序

```
事件 → [silence 静默] → [persist_filter 持续 N 点] → [dedup 去抖聚合 + 相似聚合]
     → [cooldown 冷却] → [EWMA 频率打标] → [共因归并] → [依赖抑制]
     → 规则路由复核 → i18n 渲染 → 分发（escalation 计时挂起）
```

四道闸与 L0 算法统一实现 `pkg/bellapi.Processor` 接口挂在同一条链上——**新增算法 = 接入新 Processor，不改核心**。

## 第一闸：silence（静默）

维护窗口内的事件直接入库标记 `silenced`，不产生通知。

- **匹配器**：`match`（source/type/tags 子集匹配，同规则语法）+ 时间窗——绝对窗口（`starts_at ~ ends_at`）或**每日周期窗口**（如 `23:00-06:00`，支持跨午夜）。
- **创建途径**：REST API（含按 `dedup_key` 的一键静默）、Web UI 静默按钮、Scout 通知操作。
- **恢复事件不受静默影响**——恢复必须可见，否则静默期结束会有一波假告警。

## 第二闸：dedup（去抖聚合）

窗口内相同 `dedup_key` 的事件合并为**一条**通知：首条即发（低延迟），重复事件计数落 `events.dedup_count`（UI 可见"该摄像头 1 小时内离线 12 次"）。

- 窗口默认 60s，规则可覆盖（`dedup_seconds`）。
- **相似聚合（simagg）**：文本相似的事件在窗口内归并——30 秒内 30 条移动侦测收敛为 1 条通知是 v0.1 的验收标准之一。

## 第三闸：cooldown（冷却）

per `(rule × source_ref)` 的 anti-flap 窗口：**已发过通知后的复发**在窗口内只更新计数不重发（与 dedup 的区别：dedup 管同簇合并，cooldown 管重发间隔）。窗口默认 5 分钟；`last_triggered_at` 落库，重启后冷却仍有效。恢复事件永不冷却。

## 第四闸：escalation（升级）

`error` 级通知发出后，若 `escalate_after`（默认 15 分钟）内未收到配对 `resolved`：

1. severity 升 `critical`，通知追加"已升级"标记；
2. 追加发送 `escalate_to` 渠道组（付费渠道只允许出现在这里）；
3. 升级链最多 1 跳。

计时持久化在 `escalations` 表——**进程重启后计时继续**；resolved 配对到达即取消。

## L0 算法层（纯 Go，永远在线）

不依赖任何模型与网络的"动态降噪"，参数全部规则级可覆盖：

| 算法 | 解决什么 | 默认参数 |
|---|---|---|
| **持续 N 点过滤** | 毛刺误报：单次采样跳变不通知 | N=3，仅对 warning 及以下生效；error/critical 永不过滤 |
| **EWMA 动态基线** | 同类事件频率突增检测（一晚掉线 50 次 = 网络问题而非 50 次独立故障） | alpha=0.3，窗口 24h |
| **相似聚合（simagg）** | 30 秒 30 条同源事件 → 1 条 | 60s 窗口，仅限同 source_ref |
| **共因归并** | 一次根因（交换机断电）派生 30 条设备告警 → 1 条"区域事件" | 60s 窗口，归并键 = `tags.zone` 或同 `tags.upstream` |
| **依赖抑制** | NVR 离线时抑制其下所有摄像头的派生告警 | 依赖拓扑 `tags.depends_on` 显式声明 |
| **令牌桶 + 日预算** | 渠道限频 + 付费渠道账单保护 | 渠道自报限速；`daily_budget` 付费渠道必填 |

## 降级与自愈

- 管道任何 stage panic → recover → 事件按"无管道模式"直发默认渠道组 + 审计事件 `pipeline_degraded`。
- AI（L1/L2）挂在 enrich 挂点上：超时 2s / 预算熔断 / 调用失败 → 直接走 L0 模板，**事件照发**。
- AI 只降不升 severity（原值记入 `ai_severity_from` 标签），且永远无权拦截事件——见 [AI 增强](ai.md)。

## 实测样例

演示实例灌入 18 条相同 `dedup_key` 的移动侦测事件，服务端日志：

```
level=INFO msg="event dropped by gate" gate=dedup reason="dedup ×2"
...
level=INFO msg="event dropped by gate" gate=dedup reason="dedup ×18"
level=INFO msg="escalation scheduled" dedup_key=mibee-nvr/connection/cam-front
level=INFO msg="escalations cancelled by resolve" dedup_key=mibee-nvr/connection/cam-front count=1
```

18 条事件 → 1 条通知（首条即发）；15 分钟升级计时挂起；恢复事件到达后自动取消。
