# AI Enhancements

The three iron laws (non-negotiable, from competitor post-mortems):

1. **Off by default**: no providers configured = pure L0, not one extra byte runs.
2. **Never a gate**: AI cannot drop an event and cannot raise severity — it can only "stick things onto" the event (summary/tags/downgrade suggestion). Remove the AI and notifications flow unchanged.
3. **2-second degradation**: timeout/failure/budget exhaustion → fall straight back to L0 templates; events and notifications are unaffected.

## Three tiers

| Tier | Implementation | Purpose | Cost | Status |
|---|---|---|---|---|
| **L0** | four gates + pure algorithms (EWMA/simagg/common-cause/dependency) | rule reflexes: dedup, aggregation, cooldown | zero | always on |
| **L1** | System One decision model (out-of-process local inference) | millisecond triage: false alarm? normal fluctuation? needs attention? | local CPU, free | ✅ implemented, off by default |
| **L2** | openai_compat analyzer (OpenAI-compatible endpoints) | generative summaries, semantic tags, downgrade suggestions | cloud API / LAN LLM | ✅ implemented, config-only |

AI conclusions land on the event as **append-only tags** (e.g. `ai_decision` / `ai_decision_confidence` / `ai_summary`); severity suggestions only ever downgrade (the original is kept in `ai_severity_from`).

## L2: OpenAI-compatible summaries (config-only)

Any OpenAI-compatible endpoint (DeepSeek / Qwen / OpenAI / remote Ollama):

```yaml
ai:
  daily_budget: 100        # shared across both tiers (UTC day; 0 = all AI off)
  providers:
    - name: deepseek
      base_url: "https://api.deepseek.com"
      api_key: "enc:..."   # enc: ciphertext supported
      model: "deepseek-chat"
      timeout_seconds: 2   # the 2-second law
```

Five built-in mechanisms (transparent to analyzers):

| Mechanism | What it does |
|---|---|
| Funnel triggering | gates + L0 sit upstream; only gated survivors reach AI — structurally squeezing call volume to single-digit percentages |
| Cluster batching | the request unit is the dedup cluster (one inference covers N merged events) |
| Semantic cache | conclusions cached by dedup_key with TTL; repeat alerts don't re-bill |
| Daily budget breaker | exhausting `ai.daily_budget` emits a `mibee-bell/ai.budget_exhausted` internal event through the normal pipeline (add a rule to be notified) |
| Feedback loop | the feedback API is in place; parameter write-back is planned for v1.0 |

Metrics: `bell_ai_calls_total{analyzer,result=ok|error|cache_hit|budget_skipped}`.

## L1: local triage (System One decision model)

**It generates no text**: given a state plus a typed question, a single forward pass returns a **calibrated-confidence** classification (false alarm / normal fluctuation / needs attention) in milliseconds, pure CPU. Inference runs **out of process** (keeping Bell `CGO_ENABLED=0` and Pi-compatible); Bell only speaks HTTP.

```yaml
ai:
  daily_budget: 100
  decision:
    enabled: true
    base_url: "http://<lan-box>:8000"   # the laya-serve endpoint
    model: multilingual                  # required for non-English alerts (multilingual checkpoint)
    timeout_seconds: 5                   # weak hosts: 5 (full payloads measure ~2s, cutting it close)
    min_confidence: 0.7                  # downgrades only allowed above this confidence
```

One-time endpoint setup (Apache-2.0 open weights, multilingual checkpoint):

```bash
pip install "laya[serve]"
git clone https://github.com/NandhaKishorM/laya && cd laya
python examples/server.py --device cpu --host 0.0.0.0 --port 8000
```

Measured triage quality (18-case targeted Chinese set, finalized criteria): 72% hits with **zero dangerous misjudgments** — every directional error carried confidence ≤0.36 and was blocked by the 0.7 threshold. **Safe downgrading = criteria wording + the confidence threshold, a two-lock system**; before production use, revalidate against your own real alert stream.

Custom triage questions: `instructions + criteria + downgrade_choices` are fully configurable (e.g. swap "shadow false alarms" for server-side "disk full / process crash"). Rules of thumb: only put categories you're **sure you don't want to be bothered by** into `downgrade_choices`; `min_confidence` is the knob between noise reduction and missed alerts.

## http_enrich: algorithm microservices in any language

The runtime extension axis (M9), no compilation: POST the event to any HTTP service, return `{"tags":{...},"summary":"..."}` and the tagging is done — tags append (never overwrite), the summary is prepended to the body, 10s timeout fails open (a failure skips, the chain continues):

```yaml
enrich:
  - name: cmdb
    url: http://steward:8080/api/enrich
    timeout_seconds: 5
```

Metric: `bell_enrich_total{instance,result}`. Same philosophy as AI: augment, never gate.

## Ordering

The tier-1 decision model runs before tier-2 cloud providers (local is cheaper); the enrich hook calls in registration order with the first success winning — L2 automatically becomes the fallback when the decision service is unavailable.
