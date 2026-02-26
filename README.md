# agentcents

**LLM API cost tracking proxy and budget enforcement for AI agents.**

Drop agentcents between your agent and any LLM provider. It tracks every call, enforces budgets, caches responses, and tells you exactly where your money is going — across cloud APIs and local models.

```
Your Agent  →  agentcents proxy (localhost:8082)  →  OpenAI / Anthropic / Ollama
```

No code changes required. Just point your LLM client at the proxy.

---

## Install

```bash
pip install agentcents
```

Pro features require a license key from [labham.gumroad.com](https://labham.gumroad.com).

---

## What to expect

**Zero configuration to get started.** Install, start the proxy, point your LLM client at it — that's it.

```
Step 1 — pip install agentcents          (one time)
Step 2 — start the proxy                 (once per session)
Step 3 — point your LLM client at it     (one header change)
Step 4 — agentcents usage                (see your costs)
```

No API keys, no accounts, no signup required for the free tier.

**Configuration is optional** — only add `~/.agentcents.toml` when you want:

| You want... | What to add |
|---|---|
| Hard budget limits | `[budgets] daily = 5.00` |
| Routing warnings when budget runs low | `[routing] threshold_pct = 80` |
| Track local Ollama power costs | `[local] gpu_watts = 40` |
| Separate costs per agent | `X-Agentcents-Tag` header on each call |

**Pricing data** syncs automatically on proxy startup — you never need to run `agentcents sync` manually unless you want to force a refresh after a provider announces new models.

**Pro license** — activate once per machine:
```bash
agentcents activate <your-key>
```
Pro features are then available immediately. No restart needed.

---

## Quick Start

**1. Start the proxy**
```bash
uvicorn agentcents.proxy:app --port 8082
```

**2. Point your LLM client at the proxy**
```python
import anthropic

client = anthropic.Anthropic(
    base_url="http://localhost:8082",
    default_headers={"X-Agentcents-Target": "https://api.anthropic.com"},
)
```

**3. Check your costs**
```bash
agentcents usage
agentcents recent
```

That's it. Every call is now tracked.

---

## Configuration

Create `~/.agentcents.toml` to configure budgets, routing, and local models.

```toml
# ~/.agentcents.toml

# ── Budgets ────────────────────────────────────────────────────────────────
[budgets]
daily   = 5.00    # hard block at $5/day across all calls
monthly = 50.00   # used by `agentcents rolling` reporting

# Per-tag daily budgets (optional)
[budgets.tags.my-agent]
daily = 1.00

[budgets.tags.research]
daily = 2.00

# ── Auto-routing ───────────────────────────────────────────────────────────
[routing]
mode           = "warn"   # "warn" — log suggestion only
                          # "swap" — silently swap model (Pro)
                          # "off"  — disable routing
threshold_pct  = 80       # trigger when X% of daily budget is used
skip_tool_use  = true     # never swap requests that use tools

# ── Local Models (Ollama) ──────────────────────────────────────────────────
[local]
gpu_watts        = 40     # your GPU/chip TDP in watts
                          # M1 Max ≈ 40W, M2 Ultra ≈ 60W, RTX 4090 ≈ 450W
electricity_rate = 0.12   # $/kWh — check your electricity bill
ollama_base_url  = "http://localhost:11434"

# ── Advisor ────────────────────────────────────────────────────────────────
[advisor]
min_saving_pct = 20       # only suggest swaps that save ≥ 20%
```

### Budget behavior

| Spend vs budget | Action |
|---|---|
| 0–80% | Normal |
| 80%+ | `⚠ ROUTING WARN` logged, `X-Agentcents-Suggest` header added |
| 100%+ | `429 budget_exceeded` returned, call blocked |

---

## Request Headers

Add these headers to your LLM client requests to control agentcents behavior.

| Header | Required | Example | Description |
|---|---|---|---|
| `X-Agentcents-Target` | **Yes** | `https://api.anthropic.com` | Provider base URL to forward to |
| `X-Agentcents-Tag` | No | `my-agent` | Group calls for cost reporting |
| `X-Agentcents-Session` | No | `agent-run-42` | Track individual agent sessions |
| `X-Agentcents-Cache` | No | `off` | Disable cache for this request |
| `X-Agentcents-Cache` | No | `exact` | Exact-match cache only, skip semantic |

### Examples

```python
# Tag calls by project
client = anthropic.Anthropic(
    base_url="http://localhost:8082",
    default_headers={
        "X-Agentcents-Target": "https://api.anthropic.com",
        "X-Agentcents-Tag":    "research-agent",
        "X-Agentcents-Session": "run-001",
    },
)

# Disable cache for a specific call
response = client.messages.create(
    model="claude-3-haiku-20240307",
    max_tokens=100,
    messages=[...],
    extra_headers={"X-Agentcents-Cache": "off"},
)
```

### Response headers

| Header | Description |
|---|---|
| `X-Agentcents-Cache: exact-hit` | Response served from exact-match cache |
| `X-Agentcents-Cache: semantic-hit` | Response served from semantic cache (Pro) |
| `X-Agentcents-Suggest: <model>` | Cheaper model suggested (routing warn) |
| `X-Agentcents-Routed: <model>` | Model was swapped to this (routing swap, Pro) |

---

## Local Models (Ollama)

Route Ollama calls through agentcents to track GPU power costs alongside cloud API costs.

**Start Ollama normally:**
```bash
ollama serve
```

**Point your Ollama client at the proxy:**
```bash
# Instead of http://localhost:11434
# Use    http://localhost:8082/ollama

curl http://localhost:8082/ollama/api/chat -d '{
  "model": "llama3:8b",
  "stream": false,
  "messages": [{"role": "user", "content": "hello"}]
}'
```

**Or use the OpenAI-compatible endpoint:**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8082/ollama/v1",
    api_key="ollama",
)
```

Power cost is estimated as:
```
cost = (inference_seconds / 3600) × gpu_watts × electricity_rate
```

Configure `gpu_watts` and `electricity_rate` in `~/.agentcents.toml`.

---

## CLI Reference

```
agentcents <command> [options]
```

### Cost reporting

```bash
agentcents usage                    # cost summary last 24h
agentcents usage --hours 168        # last 7 days
agentcents usage --tag my-agent     # filter by tag

agentcents recent                   # last 20 individual calls
agentcents recent --n 50            # last 50 calls

agentcents rolling                  # 30-day rolling spend
agentcents rolling --days 7         # 7-day rolling spend

agentcents agents                   # per-agent/session breakdown
agentcents agents --hours 48        # last 48h

agentcents local                    # local vs cloud cost comparison
```

### Live monitoring

```bash
agentcents watch                    # live tail of calls (Pro)
agentcents watch --poll 1           # refresh every 1 second
agentcents dashboard                # full TUI dashboard (Pro)
```

### Budget alerts

```bash
agentcents alerts                   # recent budget alerts
agentcents alerts --n 50            # last 50 alerts
```

### Catalog & models

```bash
agentcents models                   # list all models with pricing
agentcents sync                     # force sync pricing + chains
```

### Intelligence (Pro)

```bash
agentcents suggest                  # model swap suggestions based on usage
agentcents suggest --hours 168      # based on last 7 days
agentcents train                    # train XGBoost cost predictor
```

### License

```bash
agentcents activate <key>           # activate Pro license
agentcents deactivate               # remove Pro license
agentcents features                 # show available features
```

---

## Pro Features

| Feature | Free | Pro |
|---|---|---|
| Proxy + cost logging | ✓ | ✓ |
| Exact-match cache | ✓ | ✓ |
| Budget alerts + hard block | ✓ | ✓ |
| CLI reporting | ✓ | ✓ |
| Web dashboard | ✓ | ✓ |
| Local Ollama tracking | ✓ | ✓ |
| Semantic similarity cache | — | ✓ |
| Multi-agent TUI dashboard | — | ✓ |
| Live watch | — | ✓ |
| Model swap advisor | — | ✓ |
| Auto-routing (swap mode) | — | ✓ |
| XGBoost cost predictor | — | ✓ |

Get Pro at [labham.gumroad.com](https://labham.gumroad.com).

---

## Supported Providers

Any provider that speaks the OpenAI API format:

| Provider | Target URL |
|---|---|
| Anthropic | `https://api.anthropic.com` |
| OpenAI | `https://api.openai.com` |
| Google Gemini | `https://generativelanguage.googleapis.com` |
| OpenRouter | `https://openrouter.ai/api` |
| Groq | `https://api.groq.com/openai` |
| Ollama | via `/ollama` route (no header needed) |

---

## Sync

agentcents keeps two files updated in `~/.agentcents/`:

| File | Contents | Source |
|---|---|---|
| `models.json` | Model pricing ($/M tokens) | OpenRouter + LiteLLM |
| `chains.json` | Downgrade chains for routing | labham.com |

**These update in two ways:**

1. **Proxy startup** — if files are older than 24h, proxy fetches fresh data automatically when you run `uvicorn agentcents.proxy:app`
2. **Manual** — run `agentcents sync` any time to force an update

```bash
agentcents sync
# Syncing pricing catalog...
# Chains updated to v1.0.1
# Done.
```

**Why this matters:** Anthropic and OpenAI release new models frequently. Without syncing, agentcents may not recognize new model IDs or have accurate pricing. Run `agentcents sync` after any major provider announcement.

If sync fails (no internet, server down), agentcents falls back to the bundled `data/chains.json` and `data/fallback.json` that shipped with the package.

---

## Architecture

```
~/.agentcents.toml          — budgets, routing, local config
~/.agentcents/models.json   — pricing catalog (auto-updated)
~/.agentcents/chains.json   — downgrade chains (auto-updated)
~/.agentcents/ledger.db     — all call records (SQLite)
```

The proxy runs entirely locally. No call data leaves your machine.
Pricing data syncs from OpenRouter and LiteLLM APIs.
License validation calls `agentcents-license.labham.workers.dev`.

---

## License

Copyright (c) 2026 Labham LLC. All rights reserved.
Licensed under the [Labham Commercial License](https://labham.com/license).
