# LLM Provider Scenarios — Three Setups Walked End to End

> **What this is:** three concrete walkthroughs for pointing Hermes at an LLM, each tested on a real install on 2026-04-18 — including what went wrong. If you've read [Before You Start](01-installation.md#before-you-start), this doc is the deep-dive version.

## Which scenario should I pick?

| Consideration | 1. Ollama (local) | 2. OpenRouter (cloud) | 3. OpenRouter + Claude Code delegation |
|---|---|---|---|
| Per-token cost | Free after hardware | Cents per session on cheap models | Cents for orchestration + your Claude Code subscription for coding |
| Needs internet | No | Yes | Yes |
| Hardware | GPU with ≥24 GB VRAM for agent-scale models | None | None |
| Privacy | Fully local | Cloud | Cloud + your Claude setup |
| Coding quality | Mid-tier (model-dependent) | High | Highest (Claude Code remains Claude Code) |
| Setup friction | Medium — context-length quirk bites | Low | Low if you already use Claude Code |

**Quick decision:**
- Already use Claude Code daily and want the best coding results? → **Scenario 3**
- Need everything local / offline? → **Scenario 1**
- Otherwise, cheapest and simplest? → **Scenario 2**

You can also mix these — see [Combining scenarios](#combining-scenarios) at the end.

---

## Scenario 1: Ollama (local)

### Prerequisites

- [Ollama](https://ollama.com/) installed and running (`ollama serve` or a systemd-managed service on port 11434)
- A model pulled that has **both** ≥64K native context AND tool-use support. Not every Ollama model supports tools — check with `ollama show <model>`. Examples that do: `qwen3-coder:30b`, `qwen2.5-coder:32b`, `llama3.1:70b`.
- A GPU with enough VRAM to hold the model plus a 64K+ KV cache (rough rule of thumb: model file size + ~25% headroom).

### The context-length trap (read this first)

**The #1 reason Ollama + agent setups fail.** Ollama serves models with a default context window based on your VRAM, and that default is almost always below Hermes's 64K minimum:

| Available VRAM | Ollama default context |
|---|---|
| < 24 GB | 4,096 tokens |
| 24–48 GB | 32,768 tokens |
| 48+ GB | 262,144 tokens |

Even if your model's native maximum is 256K, Ollama will truncate to 32K on a 32 GB GPU unless you override the runtime context. You **cannot** set context length through the OpenAI-compatible API — it must be configured server-side or baked into a Modelfile.

Three ways to override, pick one:

```bash
# Option A: Server-wide environment variable (affects all models)
OLLAMA_CONTEXT_LENGTH=65536 ollama serve

# Option B: For systemd-managed Ollama (needs sudo)
sudo systemctl edit ollama.service
# Add: Environment="OLLAMA_CONTEXT_LENGTH=65536"
sudo systemctl daemon-reload && sudo systemctl restart ollama

# Option C: Bake into a per-model variant (no sudo, no server restart)
cat > /tmp/Modelfile <<'EOF'
FROM qwen3-coder:30b
PARAMETER num_ctx 65536
EOF
ollama create qwen3-coder-64k:latest -f /tmp/Modelfile
# Then reference qwen3-coder-64k:latest as your model name
```

Option C is the cleanest for a tutorial walkthrough because it's fully user-space and doesn't affect other models.

### Configure Hermes

```bash
hermes config set model.provider "custom"
hermes config set model.default "qwen3-coder-64k:latest"       # or whatever your tag is
hermes config set model.base_url "http://localhost:11434/v1"
hermes config set model.context_length "65536"
```

Bare `"ollama"` as a provider name is internally mapped to `"custom"` in Hermes's provider registry, so `"custom"` is the canonical value for local Ollama. (For Ollama *Cloud* — `ollama.com/v1` — use `"ollama-cloud"`; different path.)

### Verify

```bash
hermes chat -q "What is 17 * 23? Answer with just the number."
```

Expected: `391` in 5–15 seconds (varies with model and GPU).

### What we verified (2026-04-18)

Tested on an RTX 5090 (32 GB VRAM) with `qwen3-coder:30b` + a 64K Modelfile variant: `391` returned in 9 seconds. Tool-use worked correctly. A direct attempt with `qwen3:14b` (native context 40,960) was correctly rejected by Hermes for being below the 64K floor.

### Tradeoffs and when this fits

- **Fully offline** once the model is pulled
- **No marginal cost** per turn
- **Data stays on your machine** — best privacy profile of the three
- **Slower per-turn** than cloud options (~5–15s vs ~3–5s)
- **Quality plateau** — local models are generally weaker than frontier cloud models at multi-step agent work; tool-use reliability is the common failure mode
- **Hardware requirement is real** — CPU-only inference is too slow for interactive agent use

---

## Scenario 2: OpenRouter (cloud)

### Prerequisites

- [OpenRouter](https://openrouter.ai/) account and a key starting with `sk-or-v1-...`
- A small starting balance if you want reliability. Free-tier models exist but are rate-limit-prone (see below). A few dollars covers thousands of cheap-model turns.

### Configure Hermes

```bash
hermes config set OPENROUTER_API_KEY "sk-or-v1-..."
hermes config set model.provider "openrouter"
hermes config set model.default "mistralai/mistral-nemo"     # cheapest reliable; see picks below
```

### Picking a model

OpenRouter currently offers ~17 free-tier models with ≥64K context and tool-use, and many cheap paid models. Our picks:

- **`mistralai/mistral-nemo`** — cheapest paid option with tool-use and ≥64K context at the time of writing. A single verification turn is fractions of a cent. Well-suited to Hermes's orchestration-style work.
- **`qwen/qwen3-235b-a22b-2507`** — flagship Qwen3 MoE, stronger tool-use than smaller models. Slightly more expensive but more reliable for complex agent flows. Recommended if you plan to use [scenario 3 (delegation)](#scenario-3-openrouter--claude-code-delegation).
- **Free-tier candidates** — `qwen/qwen3-coder:free`, `google/gemma-4-31b-it:free`, `qwen/qwen3-next-80b-a3b-instruct:free`, and a dozen others. Pay nothing, but expect occasional HTTP 429 rate limits.

To discover what's currently available with the right capabilities, you can query OpenRouter's models API:

```bash
curl -s https://openrouter.ai/api/v1/models | python3 -c "
import json, sys
for m in json.load(sys.stdin).get('data', []):
    p = m.get('pricing', {})
    ctx = m.get('context_length', 0)
    tools = 'tools' in (m.get('supported_parameters') or [])
    if ctx >= 64000 and tools:
        print(f\"{m['id']:60s}  ctx={ctx}  in={p.get('prompt','?')}  out={p.get('completion','?')}\")
" | sort -k2 -n
```

### Verify

```bash
hermes chat -q "What is 17 * 23? Answer with just the number."
```

Expected: `391` in 3–5 seconds.

### What we verified (2026-04-18)

- `mistralai/mistral-nemo` returned `391` in 5 seconds. Real cost: ~$0.00005.
- `qwen/qwen3-coder:free` returned `HTTP 429: "qwen/qwen3-coder:free is temporarily rate-limited upstream at Venice"`. Your key is fine; the free-tier pool is shared and frequently full.

Hermes also logged a useful warning on first OpenRouter turn: *"No auxiliary LLM provider configured — context compression will drop middle turns without a summary."* This is expected when you set a primary but haven't configured the separate "compressor" model that Hermes uses to summarize context during long sessions. For short verification turns it doesn't matter; for long agent sessions, consider setting `auxiliary.provider` and `auxiliary.model` too.

### Known issue: free-tier rate limits

OpenRouter's `:free` suffix models route through third-party inference providers (Venice, SambaNova, etc.) that rate-limit the free pool aggressively. For casual CLI use you'll usually get through; for agent work (multiple tool calls per user turn), expect to hit 429s and retry. If you need reliability, add a cheap paid model as a fallback — Hermes's credential pool handles this automatically, see [Combining scenarios](#combining-scenarios).

### Tradeoffs and when this fits

- **Lowest setup friction** of the three scenarios
- **Thousands of turns for a few dollars** on cheap paid models
- **Wide model selection** (hundreds of models across providers)
- **Hermes's own tokens go on OpenRouter's bill**, which preserves your Claude Code subscription for actual coding work
- **Requires an account with payment on file** if you want reliability

---

## Scenario 3: OpenRouter + Claude Code delegation

### The idea

Hermes runs on a cheap OpenRouter model for orchestration (chat, planning, scheduling, tool-use). When coding work is heavy, Hermes spawns `claude -p "..."` as a subprocess; that subprocess runs against your Claude Code subscription as a *first-party* Anthropic client. You get Claude's coding quality without the third-party-app Extra-usage wall that hits Hermes when it calls Anthropic directly (see [chapter 1](01-installation.md#i-already-use-claude-code--do-i-need-a-separate-key)).

This is the strongest complementary case for running Hermes if you already use Claude Code daily.

### Prerequisites

- Everything from [Scenario 2](#scenario-2-openrouter-cloud) (OpenRouter key, Hermes configured for `provider: openrouter`)
- `claude` CLI installed and authenticated (`claude auth status` should show `"loggedIn": true`)
- Hermes `pty` extra installed — `uv pip install -e ".[pty]"` in your Hermes checkout
- `tmux` installed (only for interactive delegation mode; print mode doesn't need it)
- Bundled `claude-code` skill in place — check `ls ~/.hermes/skills/autonomous-ai-agents/claude-code/SKILL.md`

### Why the billing works

Run `claude auth status` and look for the `apiProvider` field:

```json
{
  "authMethod": "claude.ai",
  "apiProvider": "firstParty",
  "subscriptionType": "max"
}
```

**`apiProvider: "firstParty"` is Anthropic's own server-side designation for "this is Claude Code, route against subscription."** The key facts:

- When Hermes calls Anthropic's API **directly** using your imported OAuth token, there is no first-party identification in the request. Anthropic routes the call to your Extra-usage balance, which hits the "Add more at claude.ai/settings/usage" wall if unfunded.
- When Hermes **spawns `claude -p "..."` as a subprocess**, the `claude` binary itself handles the request, identifies as first-party, and the call routes to your subscription quota — the same way a direct `claude -p "..."` typed in your terminal does.

Running `claude auth status` on your own install is a one-command sanity check: if `apiProvider` shows `firstParty`, delegation through the subprocess will draw on subscription.

### Picking the orchestrator model

**This is the most important decision in scenario 3.** The orchestrator (the OpenRouter model Hermes itself runs on) needs to reliably invoke the terminal tool to spawn the subprocess. Small models often fail at this step.

- **`qwen/qwen3-235b-a22b-2507`** (flagship Qwen3 MoE) — recommended. Tool-use is reliable; end-to-end delegation works.
- **`openai/gpt-oss-120b`** — also solid for tool-use.
- **Smaller models (12–30B)** — often fail. They see "use the terminal tool" and just echo markdown-like syntax back without calling the tool. Avoid for scenario 3 specifically; fine for scenario 2 orchestration of direct conversation.

Set it up:

```bash
hermes config set model.default "qwen/qwen3-235b-a22b-2507"
# Keep provider: openrouter and OPENROUTER_API_KEY from scenario 2
```

### Verify delegation

```bash
hermes chat -q "Use the terminal tool to run: claude -p 'What is 17 * 23? Reply with only the number.' — then report exactly what claude printed."
```

Expected: Hermes calls the terminal tool, the subprocess runs, `claude` responds with `391`, Hermes reports it back. End-to-end: ~10–15 seconds.

### What we verified (2026-04-18)

- **Small orchestrator failed:** `mistralai/mistral-nemo` (12B) did not invoke the terminal tool. Hermes reported `0 tool calls` — the model just echoed markdown syntax and exited.
- **Flagship orchestrator succeeded:** `qwen/qwen3-235b-a22b-2507` called the terminal tool, spawned `claude -p "..."`, parsed the `391` from claude's output, and reported back. End-to-end 12 seconds.
- **Billing direction confirmed** via `apiProvider: "firstParty"` in `claude auth status`. The verification turn was also run against a subscription at 14% weekly usage, 0$ Extra usage; after the run, both displays read the same — consistent with a tiny subscription-backed call (too small to register in percent granularity) and with *not* hitting the Extra-usage wall (we'd have seen HTTP 400 as with direct Anthropic calls).

### Tradeoffs and when this fits

- **Best coding quality** of the three — Claude Code remains Claude Code
- **Subscription-friendly** — your Claude budget backs coding; Hermes's own turns run on cheap OpenRouter
- **Requires Claude Code to already be set up and authenticated**
- **Requires a capable orchestrator model** — silent failure mode with small models
- **Subprocess adds latency** (~5 s beyond the orchestrator's own response) — fine for multi-step tasks, overkill for trivial ones

---

## Combining scenarios

Hermes's credential pool holds multiple credentials per provider and can rotate or fall over between them. Three common mixes:

**Free-preferred OpenRouter with paid fallback**
Configure two OpenRouter credentials, set `model.default` to a `:free` model, and configure `fallback_providers` in `config.yaml` to fall through to a paid model on 429:

```yaml
fallback_providers:
  - provider: openrouter
    model: mistralai/mistral-nemo
```

**Ollama primary + OpenRouter fallback**
Useful for laptop setups where you sometimes work offline and sometimes not. Primary points at `http://localhost:11434/v1`; fallback is an OpenRouter model. When Ollama is unreachable, Hermes routes to the cloud.

**Any scenario + Claude Code delegation**
The `claude` subprocess (scenario 3's delegation) cares only about its own auth, not what model Hermes itself is running on. So scenario 1 (Ollama orchestrator) + Claude Code subprocess = "fully offline until I need Claude Code, then I reach out for just that one task." This is particularly useful if your Ollama hardware is beefy enough for orchestration but not for coding work.

See [chapter 3's `hermes auth` commands](03-cli-essentials.md#authenticate-credential-pools) for pool management, and the official [credential pools doc](https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools) for rotation strategies and fallback configuration.

---

## Switching scenarios later

All three live in `~/.hermes/config.yaml` under `model:`. Switch either via `hermes model` (interactive picker) or directly:

```bash
# To Ollama
hermes config set model.provider custom
hermes config set model.default qwen3-coder-64k:latest
hermes config set model.base_url http://localhost:11434/v1

# To OpenRouter
hermes config set model.provider openrouter
hermes config set model.default mistralai/mistral-nemo
hermes config set model.base_url ""     # clear the custom endpoint

# To Claude Code direct-use (only if you've funded Extra usage; otherwise see scenario 3)
hermes config set model.provider anthropic
hermes config set model.default claude-opus-4-7
hermes config set model.base_url ""
```

Scenario 3 uses the same config as Scenario 2 — the `claude` subprocess is triggered at runtime by the bundled skill or by an explicit terminal-tool invocation, not declared in the model config.

---

## See also

- [Chapter 1 — Before You Start](01-installation.md#before-you-start) — broader provider-choice context and the honest note on Claude Code billing
- [Chapter 8 — Claude Code Integration](08-claude-code-integration.md) — full details on delegation (print mode vs interactive, the bundled skill's orchestration)
- [Hermes vs. Claude Code](hermes-vs-claude-code.md) — whether to use Hermes at all if you already have Claude Code
- [REVIEW.md](../REVIEW.md) — verification ledger, including what was source-verified vs. live-verified vs. still pending

---

**← [Back to Table of Contents](../README.md)**
