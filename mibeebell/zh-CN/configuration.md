# 配置与安全

## config.yaml 全量参考

```yaml
server:
  listen: ":9080"
  base_url: ""            # 保留键（快照/深链拼接用）

auth:
  password_hash: ""       # bcrypt；mibee-bell hash-password 生成
                          # 为空 = 管理 API/UI 停用（摄取不受影响）

storage:
  driver: sqlite          # 唯一实现；postgres 预留（填了会 fail-fast 提示未实现）
  path: ./mibee-bell.db
  retention:
    events: 30d
    notifications: 90d
    deliveries: 30d

ingest:
  mqtt:                   # 列表：可订阅多个 broker
    - broker: "tcp://192.0.2.20:1883"
      topic_prefix: "mibee"
      username: ""
      password: "enc:..."
  webhook_tokens:         # per-source 长随机 token
    - token: "steward-xxxxxx"
      default_source: mibee-steward
    - token: "nvr-xxxxxx"
      default_source: mibee-nvr
  alertmanager: true      # POST /api/ingest/alertmanager
  pagerduty: true         # POST /api/ingest/pagerduty
  simple: true            # POST /api/ingest/simple（ntfy/Gotify 形）
  grafana: true           # POST /api/ingest/grafana
  bark_compat: true       # GET/POST /{key}/... → 路由到 bark 类渠道或默认渠道组

crypto:
  key: ""                 # base64 32 字节主密钥；环境变量 MIBEEBELL_KEY 可覆盖

render:
  default_lang: zh-CN     # 渠道未指定时的回落语言（zh-CN / en-US）
  timezone: Asia/Shanghai

mesh:                     # 空 = 单机模式（一等公民）
  node_name: "bell-a"
  bind_port: 7946
  join: ["192.0.2.11:7946"]

enrich:                   # http_enrich 处理器（外部打标微服务，免编译扩展）
  - name: cmdb
    url: http://192.0.2.30:8080/api/enrich
    headers:
      Authorization: "Bearer ..."
    timeout_seconds: 5    # 默认 10，上限 30
    stage: pre_dispatch   # 默认；pre_route / post_route 可选

ai:
  daily_budget: 100       # L1/L2 共享日预算（0 = AI 全关；触顶发内部事件走管道）
  providers:              # L2 openai_compat（OpenAI 兼容端点，配置即用）
    - name: deepseek
      base_url: "https://api.deepseek.com"
      api_key: "enc:..."
      model: "deepseek-chat"
  decision:               # L1 本地决策模型（默认关闭）
    enabled: false
    base_url: "http://192.0.2.30:8000"
    model: multilingual
    timeout_seconds: 2
    min_confidence: 0.7

# 首启 seed（DB 为空时导入一次，此后以 DB 为准——ADR 边界见「架构」）
rules: [...]
channels:
  wecom-ops: { type: wecom_robot, key: "enc:..." }
```

配置加载：内置默认值 → config.yaml → DB 运行态（rules/channels）覆盖合并；启动全量校验（时长格式 / 渠道 type 存在性 / 规则引用的渠道存在），错误**拒启**（fail-fast）。

## 子命令

| 命令 | 用途 |
|---|---|
| `mibee-bell -config config.yaml` | 启动 |
| `mibee-bell hash-password` | 生成 bcrypt 管理密码哈希 |
| `mibee-bell encrypt-config` | 生成/迁移凭证加密（AES-256-GCM，`enc:` 前缀自描述；幂等） |
| `mibee-bell export` | 规则/渠道反向导出 YAML（备份/迁移） |
| `mibee-bell -version` | 版本 |

## 安全模型

**管理面**

- 单管理员 `admin` + bcrypt `auth.password_hash`；登录发 JWT（12 小时）；所有 `/api/v1/*` 需 `Authorization: Bearer`。
- 登录限速与锁定（防爆破）。

**摄取面**

- 通用 webhook 走 per-source 长随机 token（路径携带），与 JWT 体系完全分离——token 只授"投递事件"权限，拿到也改不了配置。
- 兼容入口（Alertmanager / PagerDuty / simple / Grafana / Bark）设计为**内网暴露面**，按需开启，勿直接暴露公网；需要公网接入时用通用 webhook 的 token 入口。

**SSE**

- EventSource 不能带 header → 登录后换 5 分钟短期 `stream_token`（`GET /api/stream?token=`），可刷新——不引入内网免鉴权模式。

**凭证加密**

- 渠道 secret/token 以 AES-256-GCM 加密落库/落配置（`enc:<base64(nonce+ct)>`）；主密钥 `crypto.key` 或环境变量 `MIBEEBELL_KEY`。
- 无密钥 = 明文直通（向后兼容）；已加密库缺/错密钥启动即 fail-fast。
- 日志与 API 返回一律 `***` 脱敏。

**威胁模型声明**

断网、官方域名失效、运营方消失三种情况下，全部功能依然完整可用——任何路径不出现对运营方服务的强制依赖。这是"真自托管"的定义（设计铁律 1）。
