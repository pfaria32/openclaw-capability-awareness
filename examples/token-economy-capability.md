---
capability_id: token-economy
version: 1.0
triggers:
  keywords: [cost, tokens, model, routing, budget, economy, spend, expensive]
  session_types: [main, isolated]
  confidence_threshold: 0.6
priority: 10
token_budget: 600
---

# Token Economy Hooks

**Active optimizations in this fork:**

## Model Routing (before_model_select)
- GPT-4o (cheap) → Sonnet → Opus (expensive) based on task complexity
- Automatic escalation for complex tasks
- Config: `routing.cheapFirst: true`
- Hook: `before_model_select`

## Context Bundling (before_context_build)
- Hard cap: 10k tokens per context bundle
- Filters bootstrap files to fit budget
- Logs decisions to token audit trail
- Hook: `before_context_build`

## Zero-Token Heartbeat
- Pre-LLM check: if HEARTBEAT.md empty → skip API call entirely
- Saves ~50% of token spend (heartbeats are frequent)
- 100% heartbeat cost elimination when file empty
- No LLM call = $0.00 cost

**Impact:** 60-80% token reduction, ~$60-105/month savings

**Config location:** `~/.openclaw/openclaw.json`  
**Audit logs:** `~/.openclaw/token-audit/*.jsonl`  
**More info:** See TOKEN_ECONOMY.md in repo root
