# Benchmarks

> All numbers below come from **TypeSafe AI's published evaluations** unless otherwise noted. No large-scale independent reproduction has surfaced as of September 2026. Treat these as vendor-reported directional evidence, not settled ground truth.

## Latency comparison

| Model Class | End-to-End Latency | Source |
|-------------|-------------------|--------|
| Frontier LLM (GPT-5.6, Opus 5) | 3 - 329 seconds | [LLM Benchmarks](https://llm-benchmarks.diegoromero.es/) |
| Budget LLM (Terra) | ~10 seconds | TypeSafe |
| **Jev** | **70 - 500 milliseconds** | TypeSafe |

**Speedup:** 40x - 200x faster than frontier models for System One-shaped queries.

## Cost comparison

| Model | Input / MTok | Output / MTok | Notes |
|-------|-------------|--------------|-------|
| Frontier LLM | $0.20 - $10 | ~5x input | Standard pricing |
| GPT-5.6 Sol | $0.0304 / case (avg) | — | Workflow eval |
| Claude Opus 5 | $0.1761 / case (avg) | — | Workflow eval |
| **Jev** | **$0.042** | **Free** | Too cheap to meter |

**Cost advantage:** 40x - 400x cheaper per decision case.

## Workflow accuracy evals

**Source:** TypeSafe's [workflow eval dashboard](https://evals.typesafe.ai/), 711 total cases across 4 production workflows.

| Workflow | N | Jev | Best LLM | Gap |
|----------|---|-----|----------|-----|
| Security incidents | 240 | 61.7% / $0.0001 / 0.3s | Opus 66.2% / $0.0574 / 15.1s | -4.5% accuracy, 99.8% cheaper, 50x faster |
| Agent trace observability | 117 | 71.6% / $0.0003 / 0.5s | Sol 76.6% / $0.0575 / 40.3s | -5.0% accuracy, 99.5% cheaper, 80x faster |
| Invoice processing | 150 | 61.8% / $0.0011 / 0.5s | Sol 79.1% / $0.2152 / 34.3s | -17.3% accuracy, 99.5% cheaper, 68x faster |
| Customer service | 204 | 76.0% / $0.0001 / 0.4s | Sol 78.3% / $0.0323 / 10.1s | -2.3% accuracy, 99.7% cheaper, 25x faster |
| **Aggregate** | **711** | **67.8% / $0.0004 / 0.4s** | **Sol 74.1% / $0.0836 / 23.3s** | **-6.3% accuracy, 99.5% cheaper, 58x faster** |

### Key takeaways

- Jev is effectively tied with GPT-5.6 Terra (67.9%) on aggregate accuracy
- 5-6 point gap to peak accuracy models (Sol, Opus 5) on complex reasoning tasks
- On customer service (structured classification), Jev nearly matches Sol (76.0% vs 78.3%)
- On invoice processing (multi-step reasoning), Jev trails significantly (61.8% vs 79.1%)
- **Pattern:** Jev excels at classification/routing, weaker on multi-step reasoning

## Context compression

**Source:** Community report (Reddit, r/hermesagent, September 2026).

| Engine | Context Reduction | User/Agent Messages | Tool Results |
|--------|-------------------|---------------------|--------------|
| Default `ContextCompressor` | 55% | Preserved | Summarized |
| **Jev Context Engine** | **75%** | **Preserved verbatim** | **Curated by relevance** |

**20 additional percentage points of compression** with better fidelity — only old tool results are summarized, never user or agent messages.

## Real-world cost model

**Scenario:** 40,000 decisions/month intake workflow.

| Route | Monthly Cost | Notes |
|-------|-------------|-------|
| **Jev** | **$1.34** | Published pricing |
| Budget LLM | ~$90 | Estimated from $0.0304/case |
| Frontier LLM | ~$900 | Estimated from $0.0836/cave |

**Ratio:** Jev is 67x cheaper than a budget LLM route, 671x cheaper than frontier, for this volume.

## External validation (small scale)

### Every (Mike Taylor)
- 37 documents, 777 judgments in under 0.7 seconds
- Estimated cost: ~$0.0025 (quarter of a cent)
- 12 synthetic passages: Jev detected 6 of 7 writing defects vs Fable 5.1 detected all 7
- Estimated: 25x faster, 580x cheaper than Fable 5.1

### Flowtivity (browser agent)
- Browser Use agent solving 49/49 benchmark tasks
- 112x lower model cost than frontier route
- Flights found in 7 seconds for $0.0039

## What the numbers don't tell you

- **No public p95/p99 latency** under production load
- **No independent accuracy reproduction** on neutral harness
- **No long-term pricing sustainability** — current pricing may be subsidized
- **No non-English evaluation** — English-first accuracy
- **No multi-step reasoning benchmark** — Jev is weaker on chained reasoning
- **No production reliability data** (rate limits, availability, retry rates)

## When the benchmarks don't apply

Jev is **NOT** a replacement for frontier LLMs when:
- Peak accuracy on complex reasoning is required (invoice processing: -17.3%)
- Multi-step chained reasoning is needed
- The task requires explanation or rationale (Jev gives probabilities, not prose)
- Non-English language performance is critical
- The decision space is truly open-ended (not bounded options)

Jev **IS** the right choice when:
- You're making thousands of small semantic decisions (classification, routing, scoring)
- Latency matters (real-time guardrails, agent turn admission)
- Cost matters (high-volume automation)
- You need calibrated confidence for escalation logic
- The decision space is bounded (known options up front)
