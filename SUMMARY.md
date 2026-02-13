# Capability-Awareness System: Executive Summary

**TL;DR:** Make agent reliably aware of fork capabilities via smart just-in-time injection, optimized for token economy.

---

## Solution Overview

### Three-Tier System

1. **AGENTS.md (≤500 chars)** → Always loaded, points to capabilities
2. **docs/capabilities/\*.md (200-800 chars each)** → Injected when relevant
3. **SYSTEM_ARCHITECTURE.md** → Read via tool when needed (never auto-loaded)

### How It Works

```
User message
    ↓
Keyword scan (last 5 messages)
    ↓
Select relevant capability cards
    ↓
Inject into bootstrap context (max 1500 tokens)
    ↓
Agent gets targeted awareness
```

---

## Implementation Effort

**Total time:** 4-6 hours  
**Files created:** 15 new, 3 modified  
**Lines of code:** ~1200  
**Token impact:** +400 baseline, +200-600 dynamic (net neutral/positive)

---

## Quick Start

1. **Read full plan:** `CAPABILITY_AWARENESS_PLAN.md`

2. **Create structure:**

   ```bash
   mkdir -p src/plugins/capabilities
   mkdir -p docs/capabilities
   mkdir -p tests/plugins/capabilities
   ```

3. **Copy code from plan:**
   - Part 3.1: `selector.ts`, `registry.ts`
   - Part 3.2: `injector.ts`
   - Part 3.3: Modify `bootstrap-files.ts`

4. **Create capability cards:**
   - `token-economy.md` (600 tokens)
   - `telegram-optimizations.md` (400 tokens)
   - `memory-system.md` (500 tokens)

5. **Create AGENTS.md:**
   - Template in Part 2.2 (500 chars)

6. **Test:**
   ```bash
   npm run build && npm test
   ```

---

## Key Features

✅ **Smart selection** - Keywords + session type + relevance scoring  
✅ **Token-aware** - Hard cap at 1500 tokens (configurable)  
✅ **Secure** - Repo-controlled files, content sanitization  
✅ **Config-driven** - Enable/disable, force-include, exclude  
✅ **Fail-open** - Errors don't break agent startup  
✅ **Auditable** - Logs all injections with relevance scores

---

## Rollback Plan

**Quick disable:**

```json
// ~/.openclaw/openclaw.json
{
  "capabilities": {
    "enabled": false
  }
}
```

**Code disable:** Comment out hook in `bootstrap-files.ts`

**Full revert:** `git revert <commit-hash>`

---

## Token Economics

**Baseline cost:** +400 tokens (AGENTS.md + minimal capabilities)

**Dynamic cost:** +200-600 tokens (keyword-triggered)

**Net impact:** Neutral or positive

- Smarter context = better model routing = fewer retries
- Less "forgot this feature" = fewer clarification turns

**Budget control:**

```json
{
  "capabilities": {
    "maxTokens": 1500 // Adjust as needed
  }
}
```

---

## Recommended Approach

**Primary:** Just-in-time injection (as designed)

**Enhancements:**

1. Config-driven base profiles (minimal/full)
2. Add `token_economy_status` tool (complementary)
3. Keep SYSTEM_ARCHITECTURE.md for deep dives

**Alternative considered:** Capability discovery tool → ❌ Too slow, unreliable

---

## Example Capability Card

```markdown
---
capability_id: token-economy
triggers:
  keywords: [cost, tokens, model, routing, budget]
  session_types: [main, isolated]
  confidence_threshold: 0.6
priority: 10
token_budget: 600
---

# Token Economy Hooks

**Active optimizations:**

- Model routing: GPT-4o → Sonnet → Opus
- Context bundling: 10k token hard cap
- Zero-token heartbeat: Skip API if HEARTBEAT.md empty

**Impact:** 60-80% token reduction, ~$60-105/month savings
```

---

## Next Steps

1. ✅ Read full plan: `CAPABILITY_AWARENESS_PLAN.md`
2. Implement Phase 1: Infrastructure (2 hours)
3. Implement Phase 2: Injection logic (1.5 hours)
4. Create capability cards (1.5 hours)
5. Test and tune (1 hour)
6. Create PR
7. Monitor production for 48 hours
8. Iterate based on usage

---

## Questions?

**Full details:** See `CAPABILITY_AWARENESS_PLAN.md`

**Sections:**

- Part 1: System survey
- Part 2: File/folder layout
- Part 3: Just-in-time injection implementation
- Part 4: Implementation plan with code touchpoints
- Part 5: Rollback plan
- Part 6: Alternative approaches (with evaluation)
- Part 7: Recommendations
- Part 8: Quick start

**Appendix:** Example capability cards (telegram, memory, second-brain)
