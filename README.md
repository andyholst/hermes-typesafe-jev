# Hermes + Jev: System One Decision Layer for AI Agents

> Typed, probabilistic decisions at 100ms latency — 40-400x cheaper than frontier LLMs for classification, routing, and scoring.

This is the documentation hub for integrating **[Jev](https://typesafe.ai)** (TypeSafe AI's System One Model) with **[Hermes Agent](https://hermes-agent.nousresearch.com)** (Nous Research's autonomous AI agent framework).

## What is Jev?

Jev is a **System One Model** — a class of AI model built for **fast, structured decisions** instead of text generation. While traditional LLMs generate tokens sequentially (great for chat, terrible for `if` statements), Jev takes application state + typed questions and returns **probabilistic decisions with calibrated confidence** in 70-500ms.

| | Frontier LLM (GPT-5.6, Opus 5) | Jev (System One) |
|---|---|---|
| **Latency** | 3-329 seconds | 70-500 milliseconds |
| **Input cost** | $0.20-$10 / MTok | $0.042 / MTok |
| **Output cost** | ~5x input | **Free** |
| **Confidence** | Overconfident, inconsistent | Calibrated probability |
| **Error modes** | Hallucination, refusal, parse failures | Cannot violate schema |

## How it works with Hermes Agent

Hermes has **native Jev integration** built into its core — no plugin required for basic usage. The `hermes-jev` community plugin extends this with a full context engine, nervous system, and adaptive routing.

### Built-in Jev tools (Hermes core)

Hermes exposes 8 `jev_*` tools directly in every session:

| Tool | Purpose |
|------|---------|
| `jev_assess` | Ask Jev questions about the current agent state (choice/score/noul) |
| `jev_decide` | Make one bounded typed decision with TypeSafe Jev |
| `jev_rank` | Rank a bounded set of candidate labels using Jev probabilities |
| `jev_verify` | Verify an execution result with a bounded Jev check |
| `jev_nervous_event` | Emit a material decision into the asynchronous nervous system |
| `jev_context_curate` | Govern what context enters the LLM call (context-value governor) |
| `jev_context_rehydrate` | Restore sanitized evidence previously replaced by curation |
| `jev_stats` | Return bounded local telemetry for the active profile |

### hermes-jev plugin (community)

The [`hermes-jev`](https://github.com/keeltrace/hermes-jev) plugin by keeltrace adds:

- **Context Engine** — replaces Hermes's default `ContextCompressor` with Jev-powered context governance. Curates what enters the LLM each turn, achieving **75% context reduction** (vs 55% default) while preserving every user/agent message verbatim.
- **Nervous System** — asynchronous decision loop that monitors agent behavior, detects repeated failures, and triggers local replanning without waiting for remote supervision.
- **Turn Admission** — confidence-gated challenges before a turn is accepted.
- **Receipt-backed Verification** — every decision is logged with evidence for auditability.
- **Adaptive Routing** — chooses between local fallback and remote Jev calls based on confidence and cost.

### Architecture at a glance

```
User message
     │
     ▼
┌─────────────────────┐
│  Turn Admission     │ ← Jev assesses: is this turn worth taking?
│  (nervous system)   │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Context Selection  │ ← Jev curates what context enters the LLM
│  (context engine)   │   (75% reduction, verbatim user/agent messages)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  LLM Call           │ ← Only curated context sent to expensive model
│  (frontier model)   │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Tool Loop          │ ← jev_decide / jev_rank / jev_assess for
│  (agent actions)    │   classification, routing, scoring decisions
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Verification       │ ← jev_verify checks results before delivery
│  (receipt-backed)   │
└─────────────────────┘
```

## Benchmarks

### TypeSafe's published workflow evals (711 cases, 4 workflows)

| Workflow | N | Jev Accuracy / Cost / Latency | Best LLM Comparator |
|----------|---|-------------------------------|---------------------|
| Security incidents | 240 | 61.7% / $0.0001 / 0.3s | Opus 66.2% / $0.0574 / 15.1s |
| Agent trace observability | 117 | 71.6% / $0.0003 / 0.5s | Sol 76.6% / $0.0575 / 40.3s |
| Invoice processing | 150 | 61.8% / $0.0011 / 0.5s | Sol 79.1% / $0.2152 / 34.3s |
| Customer service | 204 | 76.0% / $0.0001 / 0.4s | Sol 78.3% / $0.0323 / 10.1s |
| **Aggregate** | **711** | **67.8% / $0.0004 / 0.4s** | **Sol 74.1% / $0.0836 / 23.3s** |

### Context compression comparison

| Engine | Context Reduction | User/Agent Messages | Tool Results |
|--------|-------------------|---------------------|--------------|
| Default `ContextCompressor` | 55% | Preserved | Summarized |
| **Jev Context Engine** | **75%** | **Preserved verbatim** | **Curated by relevance** |

### Real-world cost model (40,000 decisions/month)

| Route | Monthly Cost |
|-------|--------------|
| **Jev** | **$1.34** |
| Budget LLM | ~$90 |
| Frontier LLM | ~$900 |

> **Note:** All TypeSafe benchmarks are vendor-reported. Independent reproduction is pending. Treat accuracy parity as promising, not settled.

## What you actually get

### 1. Token savings on routine decisions
Every classification, routing, or scoring decision that would normally burn 500-2000 tokens from your frontier model now costs ~$0.0004 and completes in 0.4s. Over a 100-turn agent run, this adds up fast.

### 2. Loop breaking
The nervous system detects when an agent repeats the same failed action. After 3 identical failures, it triggers a local `REPLAN` — blocking the exact same failed action and forcing a different approach. No more watching your agent retry `curl` 50 times against a dead endpoint.

### 3. Confidence-calibrated escalation
Every Jev decision ships with a calibrated probability. Route high-confidence decisions automatically, medium-confidence to human review, and low-confidence to the frontier model. No more guessing when the model is "sure."

### 4. Context governance
The Jev context engine decides what enters the LLM context each turn. Old tool results that are stale or recoverable get replaced with compact summaries. Active constraints and failure evidence are preserved. Result: 75% less context, same or better task completion.

### 5. Fail-open design
If Jev is unavailable or times out, the system falls back to the default behavior. A failing engine is never worse than not installing one.

## Related plugins and ecosystem

| Plugin | Author | Description |
|--------|--------|-------------|
| [`hermes-jev`](https://github.com/keeltrace/hermes-jev) | keeltrace | Full integration: context engine, nervous system, verification, routing |
| `jev-memory-selector` | — | Selects which memories to load using Jev |
| `jev-mcp-router` | — | Routes MCP server calls through Jev |
| `jev-typesafe` | — | TypeSafe SDK integration for Hermes |
| `jev-agent-router` | — | Agent Plugins v1: Jev-assisted agent routing |
| `jev-model-router` | — | Model routing with Jev (budget/capability constraints) |
| `jev-approvals` | — | Approval decisions via Jev |

## Getting started

### Basic (built-in tools)

No installation needed. Jev tools are available in every Hermes session:

```bash
hermes chat -q "Use jev_decide to classify this error: connection timeout on port 443"
```

### Advanced (with hermes-jev plugin)

```bash
# Install the plugin
hermes plugins install keeltrace/hermes-jev

# Activate the Jev context engine
hermes config set context.engine jev

# Enable the nervous system (turn admission + loop breaking)
hermes config set plugins.entries.hermes-jev.settings.nervous_enabled true

# Configure remote Jev calls
hermes config set OPENROUTER_API_KEY sk-or-...
# or
hermes config set TYPESAFE_API_KEY ...
```

### Verify it's working

```bash
hermes plugins list | grep jev
# │ hermes-jev │ enabled │ 0.2.1.2 │ ...
```

## Further reading

- [TypeSafe AI — System One Models](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- [Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/)
- [Hermes Context Engine Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/context-engine-plugin)
- [Hermes Hooks Reference](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md)
- [TypeSafe Workflow Evals](https://evals.typesafe.ai/)
- [Kingy AI Jev Review](https://kingy.ai/blog/typesafe-jev-review-the-ai-model-that-doesnt-generate-text/)
- [DataCamp Jev Explained](https://www.datacamp.com/blog/system-one-models-jev)
- [Flowtivity Jev Use Cases](https://flowtivity.ai/blog/jev-use-cases-vs-text-output-llms/)
