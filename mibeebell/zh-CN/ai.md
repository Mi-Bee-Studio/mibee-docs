# AI 增强

三铁律（不可妥协，来自竞品尸检）：

1. **默认关闭**：不配置 providers 即纯 L0，一个字节都不多跑。
2. **永不闸门**：AI 无权拦截事件、无权升 severity——它只能往事件上"贴东西"（摘要/标签/降级建议）。摘除 AI，通知照发。
3. **2 秒降级**：超时/失败/预算熔断 → 直接走 L0 模板，事件与通知零影响。

## 三档能力

| 档 | 实现 | 用途 | 成本 | 状态 |
|---|---|---|---|---|
| **L0** | 四道闸 + 纯算法（EWMA/相似聚合/共因归并/依赖抑制） | 规则反射：去抖、聚合、冷却 | 零 | 永远在线 |
| **L1** | System One 决策模型（进程外本地推理） | 毫秒级分诊：误报？正常波动？需关注？ | 本地 CPU，免费 | ✅ 已实现，默认关闭 |
| **L2** | openai_compat Analyzer（OpenAI 兼容端点） | 生成式摘要、语义标签、降级建议 | 云 API / 局域网 LLM | ✅ 已实现，配置即用 |

AI 结论以**追加式标签**落事件（如 `ai_decision` / `ai_decision_confidence` / `ai_summary`）；severity 建议只降不升（ADR 纪律），原值记入 `ai_severity_from`。

## L2：OpenAI 兼容摘要（配置即用）

任何 OpenAI 兼容端点（DeepSeek / 通义 / OpenAI / 远程 Ollama）：

```yaml
ai:
  daily_budget: 100        # 两档 analyzer 共享（UTC 日计数；0 = AI 全关）
  providers:
    - name: deepseek
      base_url: "https://api.deepseek.com"
      api_key: "enc:..."   # 支持 enc: 密文
      model: "deepseek-chat"
      timeout_seconds: 2   # 2 秒铁律
```

内置五机制（对 Analyzer 透明）：

| 机制 | 说明 |
|---|---|
| 漏斗分级触发 | 四闸 + L0 在链前段，只有过闸事件才到达 AI（结构性把调用量压到个位数百分比） |
| 簇级批量 | 请求单位 = dedup 簇（同簇上游已被闸合并，一次推理覆盖 N 条） |
| 语义缓存 | 按 dedup_key TTL 缓存复用结论，重复告警不重复花钱 |
| 日预算熔断 | `ai.daily_budget` 触顶发 `mibee-bell/ai.budget_exhausted` 内部事件走普通管道（配条规则即可收到提醒） |
| 反馈闭环 | feedback API 已备；参数回写规划 v1.0 |

指标：`bell_ai_calls_total{analyzer,result=ok|error|cache_hit|budget_skipped}`。

## L1：本地分诊（System One 决策模型）

**不生成文本**：给一个状态 + 类型化问题，单次前向返回**校准过置信度**的分类决策（误报 / 正常波动 / 需关注），毫秒级、纯 CPU。模型推理在**进程外**跑（保持 Bell `CGO_ENABLED=0` 与树莓派兼容），Bell 只发 HTTP。

```yaml
ai:
  daily_budget: 100
  decision:
    enabled: true
    base_url: "http://<lan-box>:8000"   # laya-serve 端点
    model: multilingual                  # 中文告警必须固定（多语种 checkpoint）
    timeout_seconds: 5                   # 弱宿主建议 5（完整载荷实测 ~2s 贴边）
    min_confidence: 0.7                  # 置信度达标才允许降级建议
```

端点侧一次性部署（Apache-2.0 开放权重，多语种 checkpoint 含中文）：

```bash
pip install "laya[serve]"
git clone https://github.com/NandhaKishorM/laya && cd laya
python examples/server.py --device cpu --host 0.0.0.0 --port 8000
```

中文分诊实测（18 例定向集）：判据措辞定稿后命中 72%、**危险错判 0**——全部方向性错误置信 ≤0.36，被 0.7 门槛全数拦截。**降级安全 = 判据措辞 + 置信门槛双保险**，缺一不可；生产启用前建议用自家真实告警流扩样验证。

自定义分诊问题：`instructions + criteria + downgrade_choices` 完全可配（例如把"光影误报"换成服务器场景的"磁盘满/进程崩溃"）。要点：`downgrade_choices` 只放**确定不想被打扰**的类别；`min_confidence` 是降噪与漏报之间的旋钮。

## http_enrich：用任何语言写"算法微服务"

免编译的运行时扩展轴（M9）：POST 事件给任意 HTTP 服务，返回 `{"tags":{...},"summary":"..."}` 即完成外部打标——tags 追加不覆盖、summary 前置正文、10s 超时 fail-open（失败跳过，链继续）：

```yaml
enrich:
  - name: cmdb
    url: http://steward:8080/api/enrich
    timeout_seconds: 5
```

指标 `bell_enrich_total{instance,result}`。与 AI 同款哲学：增强不闸门。

## 挂载顺序

tier 1 决策模型先于 L2 云端 provider（本地更便宜）；enrich 挂点按注册顺序调用、首个成功者生效，L2 自动成为决策服务不可用时的回退。
