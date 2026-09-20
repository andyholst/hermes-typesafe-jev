# Hermes + Jev Architecture

Deep dive into how Jev integrates with Hermes Agent's plugin system, context engine, and nervous system.

## System One vs Traditional LLMs

### Traditional LLM (System Two)
```
Input → Reasoning → Token 1 → Token 2 → ... → Token N → Text → Parse → Software decision
```
- Sequential token generation
- Output is a string — must be parsed and validated
- Can hallucinate, refuse, or emit unparseable shapes
- 3-329 seconds end-to-end
- Expensive: $0.20-$10/MTok input, ~5x for output

### Jev (System One)
```
Application State + Typed Questions → Jev Model → Typed Decisions → Software
```
- Parallel question evaluation
- Output is a typed probability distribution — no parsing needed
- Cannot violate schema (mathematically guaranteed)
- 70-500ms end-to-end
- Cheap: $0.042/MTok input, **free output**

## Three core decision types

### 1. Choice
Select one option from a predefined set with probability distribution.

```json
{
  "question": "Which queue should receive this ticket?",
  "options": ["billing", "technical", "shipping", "human_review"],
  "response": {
    "choice": "billing",
    "probability": 0.89,
    "distribution": {"billing": 0.89, "technical": 0.07, "shipping": 0.03, "human_review": 0.01}
  }
}
```

### 2. Score
Numeric or rubric scoring with calibrated confidence.

```json
{
  "question": "How urgent is this issue? (0-10)",
  "response": {"score": 8, "confidence": 0.92}
}
```

### 3. Noul
Yes/no probability.

```json
{
  "question": "Does this request describe a duplicate charge?",
  "response": {"noul": 0.97}
}
```

## Hermes plugin architecture

### Context Engine ABC

Hermes context engines implement the `ContextEngine` ABC:

```python
from agent.context_engine import ContextEngine

class JevContextEngine(ContextEngine):
    @property
    def name(self) -> str:
        return "jev"
    
    def should_compress(self, prompt_tokens: int = None) -> bool:
        # Return True when compaction should fire
    
    def compress(self, messages: list, current_tokens: int = None,
                 focus_topic: str = None) -> list:
        # Compact message list, return new sequence
    
    def select_context(self, request_messages, *, conversation_messages=None,
                       incoming_message=None, budget_tokens=0):
        # Choose/replace context for THIS request before dispatch
    
    def on_turn_complete(self, messages, usage=None, **kwargs):
        # Observe finished turn, ingest/index for next select_context()
```

### Per-turn lifecycle

```
1. User sends message
       │
2. pre_llm_call hooks fire
       │
3. context_engine.select_context()  ← Jev curates what enters the LLM
       │
4. LLM call (only curated context sent)
       │
5. Tool loop
       │   ├── pre_tool_call hooks
       │   ├── jev_assess / jev_decide / jev_rank (agent calls)
       │   ├── tool execution
       │   └── post_tool_call hooks
       │
6. context_engine.on_turn_complete() ← Jev observes, indexes for next turn
       │
7. Verification (jev_verify)
       │
8. Final response delivered
```

### Registration

```yaml
# config.yaml
context:
  engine: "jev"    # activates the Jev context engine
```

## hermes-jev plugin internals

### 8 tools registered

| Tool | Purpose |
|------|---------|
| `jev_assess` | Ask Jev questions about current state |
| `jev_decide` | One bounded typed decision |
| `jev_rank` | Rank candidates with Jev probabilities |
| `jev_verify` | Verify execution result |
| `jev_nervous_event` | Emit material decision to nervous system |
| `jev_context_curate` | Govern context entering LLM call |
| `jev_context_rehydrate` | Restore sanitized evidence |
| `jev_stats` | Return local telemetry |

### 7 hooks registered

| Hook | Purpose |
|------|---------|
| `pre_llm_call` | Inject curated context |
| `post_llm_call` | Observe LLM response for governance |
| `pre_tool_call` | Assess tool call relevance |
| `post_tool_call` | Observe tool results |
| `transform_tool_result` | Curate result before conversation append |
| `on_session_start` | Initialize Jev state |
| `on_session_end` | Flush state, close connections |

### The nervous system

The nervous system is an **asynchronous decision loop** that runs alongside the agent:

```
┌──────────────────────────────────────────────┐
│              Nervous System                  │
│                                              │
│  Turn Admission ──→ Is this turn worth it?   │
│       │                                      │
│  Failure Detection ──→ Repeated same error?  │
│       │                                      │
│  Local Replan ──→ Try different approach     │
│       │                                      │
│  Escalation ──→ Need human/stronger model?   │
│       │                                      │
│  Receipt Log ──→ Audit trail of decisions    │
└──────────────────────────────────────────────┘
```

**Repeated failure loop breaker:**

```
Attempt 1: curl https://api.example.com/data → timeout
Attempt 2: curl https://api.example.com/data → timeout (deduplicated)
Attempt 3: BLOCKED → Local REPLAN triggered → try different approach
```

Configuration:
```bash
# How many identical failures before local replan (default: 3)
hermes config set plugins.entries.hermes-jev.settings.nervous_repeated_failure_local_replan_at 3
```

### Adaptive routing

```
Low confidence / high stakes → Remote Jev (OpenRouter Decisions)
High confidence / low stakes → Local fallback (no API call)
```

## What gets sent to Jev

With default settings (`nervous_enabled` / `turn_admission` on):

- Each turn's user prompt (up to 12k characters)
- Redacted tool/result previews
- Uses `OPENROUTER_API_KEY` or `TYPESAFE_API_KEY`

**Fail-open:** If Jev is unavailable or times out, the system falls back to default behavior. A failing engine is never worse than not installing one.

## Privacy & secrets

- Tool arguments/results are **redacted** before sending to Jev
- Only compact previews are sent (not full conversation)
- Nervous system receipts are stored locally (never sent externally)
- `jev_stats` is a local-only tool — never calls Jev/OpenRouter

## Thread safety

The context engine is identity-checked for the inherited ABC default. Non-implementing engines pay no per-request work. Hook callbacks that block longer than `plugins.hook_callback_timeout` (default 30s) are abandoned without joining the worker, so the agent loop continues.
