# Capability-Awareness System: Executive Summary (v2.0)

**TL;DR:** Router-gated just-in-time capability injection with zero baseline cost and comprehensive security.

---

## What Changed (v1.0 → v2.0)

**Critical corrections applied:**

✅ **Real temp files** (no more synthetic paths)  
✅ **Frontmatter-only registry** (no duplication)  
✅ **Router-tag selection** (no hypothetical functions)  
✅ **Content-length caps** (enforced, not estimated)  
✅ **Zero baseline cost** (router-gated only)  
✅ **Repo-only loading** (security hardened)  
✅ **Per-card validation** (non-blocking)  
✅ **Safe rollback** (proper JSON editing)  
✅ **Metrics instrumentation** (kill switch + analytics)  
✅ **Discovery tool retained** (complementary)

---

## Solution Overview

### Three-Tier System

1. **AGENTS.md (≤600 chars)** → Always loaded, points to capabilities
2. **docs/capabilities/\*.md (200-800 chars each)** → Injected when router tags match
3. **SYSTEM_ARCHITECTURE.md** → Read via tool (never auto-loaded)

### How It Works

```
User message
    ↓
Router classifies (tags: config, integration, token-economy, etc.)
    ↓
  Tags match capability triggers?
    ↓
  NO → Skip injection (0 tokens) ✅
  YES → Select relevant capabilities
    ↓
Parse frontmatter (in-memory registry)
    ↓
Calculate relevance scores
    ↓
Sort by priority
    ↓
Write to ~/.openclaw_runtime/CAPABILITIES.md (real file)
    ↓
Inject into bootstrap context
    ↓
Agent gets targeted awareness
```

---

## Token Economics (Refined)

**Baseline cost:** **0 tokens** (no injection unless router requests)

**Dynamic cost:** +200-600 tokens when router tags match

**Trigger frequency:** ~5-10% of sessions (estimated)

**Net monthly impact:** +$3-6/month

**Net benefit:** Better routing → fewer retries → net neutral or positive

**Kill switch:** Auto-disable if >30% injection rate for 24h

---

## Implementation Effort

**Total time:** 6-8 hours (up from 4-6 due to security hardening)

**Files created:** 18 new, 3 modified

**Lines of code:** ~1500

**Token impact:** 0 baseline, +200-600 dynamic

---

## Quick Start

1. **Read technical assessment:** `TECHNICAL_ASSESSMENT.md`
2. **Read refined plan:** `REFINED_IMPLEMENTATION_PLAN.md`
3. **Follow phases:** Infrastructure → Injection → Integration → Content
4. **Test thoroughly:** Unit + integration + E2E
5. **Deploy with monitoring:** Track metrics for 48 hours

---

## Key Features (Refined)

✅ **Router-gated selection** - Zero cost unless router tags match  
✅ **Real temp files** - No synthetic paths, guaranteed to work  
✅ **Frontmatter-only registry** - Single source of truth (no drift)  
✅ **Hard token caps** - Content-length enforced (not estimated)  
✅ **Repo-only loading** - Security-first (workspace opt-in only)  
✅ **Per-card validation** - Non-blocking (skip bad cards, not all)  
✅ **Safe rollback** - Proper JSON editing (no corruption)  
✅ **Metrics instrumentation** - Kill switch + analytics  
✅ **Discovery tool** - Complementary debugging

---

## Example Capability Card (Refined)

```markdown
---
capability_id: token-economy
name: Token Economy Hooks
triggers:
  routerTags: [token-economy, cost, budget, routing]  # Primary
  keywords: [cost, tokens, model, routing]             # Fallback
  sessionTypes: [main, isolated]
  confidenceThreshold: 0.6
priority: 10
max_chars: 2400      # Hard cap (enforced)
token_budget: 600    # Estimate (for logging)
---

# Token Economy Hooks

**Active optimizations:**
- Model routing: GPT-4o → Sonnet → Opus
- Context bundling: 10k token hard cap
- Zero-token heartbeat: Skip API if HEARTBEAT.md empty

**Impact:** 60-80% token reduction

**Check status:**
```bash
openclaw token-economy status
```
````

---

## Security (Hardened)

**File Access:**
- Default: Repo-only (`docs/capabilities/`)
- Workspace fallback: Opt-in only (disabled by default)
- Path validation: Realpath check + symlink rejection + size caps

**Content Validation:**
- Per-card sanitization (non-blocking)
- Suspicious pattern detection
- XML role tag stripping

**Rollback:**
- Safe JSON editing (no cat >> corruption)
- Backup before modify
- Service-name verified

**Audit Logging:**
```
[capability-injection] Injected 2 capabilities (850 tokens):
  token-economy (1.00), telegram-optimizations (0.62)
```

---

## Configuration (Refined)

```json
{
  "capabilities": {
    "enabled": true,
    "maxTokens": 1500,
    "allowWorkspaceCards": false,        // Security: Default false
    "forceInclude": ["token-economy"],
    "exclude": [],
    "autoDisableThreshold": 0.30,        // Kill switch
    "autoDisableWindowHours": 24
  }
}
```

**Quick disable:**
```bash
node scripts/disable-capabilities.js
docker compose restart openclaw-gateway
```

---

## Rollback Plan (Safe)

**Level 1: Config (instant)**
```bash
node scripts/disable-capabilities.js  # Safe JSON editing
docker compose restart
```

**Level 2: Code (5 min)**
```typescript
// Comment out injection hook in bootstrap-files.ts
npm run build && docker compose restart
```

**Level 3: Git (10 min)**
```bash
git revert <commit-hash>
npm run build && docker compose restart
```

---

## Discovery Tool (Complementary)

**Added (not replaced):**

```bash
# List available capabilities
openclaw list-capabilities

# Full details
openclaw list-capabilities --detail=full
```

**Use cases:**
- Agent debugging: "What capabilities should I have?"
- User debugging: "Is token-economy enabled?"
- Status checks: "Which capabilities are active?"

---

## Metrics & Monitoring

**Logged to:** `~/.openclaw/capability-metrics.jsonl`

**Metrics tracked:**
- Injection rate (% sessions)
- Avg tokens per injection
- Most frequently injected capabilities
- Selection time (performance)

**Kill switch:**
- Threshold: >30% injection rate for 24h
- Action: Auto-disable + alert
- Minimum samples: 50 sessions

**Daily report:**
```bash
cat ~/.openclaw/capability-metrics.jsonl | \
  jq -s 'map(select(.timestamp > (now - 86400))) | 
         {triggered: map(select(.totalTokens > 0)) | length, 
          total: length, 
          rate: (map(select(.totalTokens > 0)) | length) / length}'
```

---

## Recommended Architecture

**For OpenClaw forks:**

1. **Always-On (Tier 1):**
   - Tight AGENTS.md (≤600 tokens)
   - Reference to discovery tool
   - No automatic injection

2. **On-Demand Injection (Tier 2):**
   - Triggered by router tags
   - Real file (`.openclaw_runtime/CAPABILITIES.md`)
   - Strict path validation
   - Repo-only loading

3. **Complementary Tools (Tier 3):**
   - `list_capabilities` - Discovery
   - `token_economy_status` - Stats
   - `env_presence_check` - Validation

4. **Single Source of Truth:**
   - Markdown frontmatter (parsed at startup)
   - No separate TS registry

---

## Required Changes Checklist

**Before implementation:**

- [x] Synthetic file → Real temp file
- [x] Registry duplication → Frontmatter only
- [x] getRecentMessages → Router tags
- [x] Token estimates → Content-length caps
- [x] Baseline injection → Router-gated only
- [x] Workspace fallback → Disabled by default
- [x] Sanitization → Per-card, non-blocking
- [x] Rollback → Safe JSON editing
- [x] Metrics → Instrumentation added
- [x] Discovery tool → Retained as complementary

---

## Success Criteria

**Before PR approval:**

- [ ] All unit tests pass
- [ ] Integration test passes
- [ ] E2E test with real gateway passes
- [ ] Manual testing confirms:
  - Zero baseline cost (no injection without router tags)
  - Injection works when router tags match
  - Kill switch triggers correctly
  - Rollback script works
  - Security controls effective
- [ ] Token impact measured:
  - Baseline: 0 tokens ✅
  - Dynamic: +200-600 tokens when triggered
- [ ] Documentation complete
- [ ] Security validated
- [ ] Metrics instrumentation functional

---

## Next Steps

1. ✅ Technical assessment complete
2. ✅ Refined implementation plan complete
3. ▶️ **Implement Phase 1** (Infrastructure: types, registry, sanitizer)
4. Implement Phase 2 (Selection & injection logic)
5. Implement Phase 3 (Hook integration)
6. Implement Phase 4 (Capability cards & docs)
7. Test (unit + integration + E2E)
8. Create PR
9. Deploy to production
10. Monitor for 48 hours
11. Iterate based on metrics

---

## Final Recommendation

**Direction:** ✅ **Approved with corrections applied**

**Confidence:** High (all critical issues addressed)

**Token impact:** Net neutral or positive

**Security:** Hardened (repo-only, validated paths, per-card sanitization)

**Maintainability:** High (single source of truth, content-only updates)

**Rollback safety:** Excellent (config + code + git levels)

---

**Status:** Ready for implementation

**Estimated ROI:**
- Dev time: 6-8 hours
- Monthly cost: +$3-6 (minimal)
- Benefit: Reliable capability awareness + better routing + fewer retries
- Net: Positive (intangible reliability gains + tangible routing improvements)

---

**Questions?**

**Full details:**
- `TECHNICAL_ASSESSMENT.md` - What was wrong and why
- `REFINED_IMPLEMENTATION_PLAN.md` - How to implement correctly
- `ARCHITECTURE.md` - Design decisions and tradeoffs

**For implementation:** Follow `REFINED_IMPLEMENTATION_PLAN.md` Phase 1-4
