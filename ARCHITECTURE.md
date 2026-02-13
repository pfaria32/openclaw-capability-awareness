# Capability-Awareness System: Architecture & Design Decisions

**Version:** 2.0 (Post-Assessment Refinement)  
**Date:** 2026-02-13  
**Status:** Approved for Implementation

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Design Goals](#2-design-goals)
3. [Core Architecture](#3-core-architecture)
4. [Key Design Decisions](#4-key-design-decisions)
5. [Security Model](#5-security-model)
6. [Token Economics](#6-token-economics)
7. [Alternative Approaches](#7-alternative-approaches)
8. [Tradeoffs](#8-tradeoffs)
9. [Extension Points](#9-extension-points)
10. [Future Considerations](#10-future-considerations)

---

## 1. Problem Statement

### 1.1 The Core Issue

**Scenario:** You fork an AI agent framework (OpenClaw) and add custom capabilities:
- Token-economy hooks (model routing, context bundling)
- Platform optimizations (Telegram, Discord)
- Integration modules (RAG memory, Second Brain)
- Security controls (OpenClaw Shield)

**Problem:** Agent doesn't reliably know about these capabilities.

**Why?**
- Custom features aren't in the base framework's prompt
- Adding them to always-on prompt costs tokens (every session)
- Updating capabilities requires prompt changes (fragile)
- No structured way to document fork-specific features

### 1.2 Requirements

**Must have:**
1. Agent reliably aware of fork capabilities
2. Zero baseline token cost (token-economy compatible)
3. Dynamic awareness based on conversation context
4. Secure (no prompt injection surface)
5. Maintainable (content updates don't require code changes)
6. Safe rollback (disable without breaking agent)

**Nice to have:**
7. Discovery tool for debugging
8. Metrics for monitoring
9. Kill switch for runaway costs
10. Extensible for future capabilities

---

## 2. Design Goals

### 2.1 Primary Goals

1. **Reliable Awareness:** Agent knows when to leverage capabilities
2. **Token Efficiency:** Zero cost unless needed
3. **Maintainability:** Content-only updates (no code changes)
4. **Security:** No injection surface from user-editable files

### 2.2 Non-Goals

- ❌ Replace skills system (complementary, not replacement)
- ❌ Dynamic capability installation (repo-controlled only)
- ❌ User-editable capabilities (security risk)
- ❌ Real-time capability discovery (too slow)

### 2.3 Success Metrics

**Token efficiency:**
- Baseline: 0 tokens (✅ target met)
- Dynamic: <1500 tokens when triggered
- Frequency: <15% of sessions

**Reliability:**
- Agent uses capabilities when available: >90%
- False positives (unnecessary injection): <5%

**Maintainability:**
- Time to add capability: <30 min
- Time to update capability: <10 min
- No code changes for content updates

---

## 3. Core Architecture

### 3.1 System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      User Message                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │   Router      │  Classifies message
              │ Classification│  Adds tags: config, integration, etc.
              └───────┬───────┘
                      │
                      ▼
         ┌────────────────────────┐
         │ Capability Injection   │  Router tags → selection
         │  (before_context_build)│
         └────────┬───────────────┘
                  │
                  ├─ NO MATCH → Skip injection (0 tokens)
                  │
                  └─ MATCH → Select capabilities
                             │
                             ▼
                  ┌──────────────────┐
                  │ Registry Builder │  Parse frontmatter (in-memory)
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Selector        │  Score relevance, sort by priority
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Sanitizer       │  Validate & clean content (per-card)
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Injector        │  Write real temp file
                  │                  │  ~/.openclaw_runtime/CAPABILITIES.md
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │ Bootstrap Context│  Inject into agent context
                  └────────┬─────────┘
                           │
                           ▼
                     ┌──────────┐
                     │  Metrics │  Log usage, check kill switch
                     └──────────┘
```

### 3.2 Three-Tier Model

**Tier 1: AGENTS.md (Always Loaded)**
- Ultra-concise (≤600 chars)
- Points to capabilities (doesn't describe)
- References discovery tool
- Minimal token cost

**Tier 2: Capability Cards (Just-In-Time)**
- Located: `docs/capabilities/*.md`
- Triggered by router tags
- Frontmatter metadata + markdown body
- 200-800 chars each

**Tier 3: Deep Reference (On-Demand)**
- `SYSTEM_ARCHITECTURE.md`
- Agent reads via tool when needed
- Never auto-injected

### 3.3 Data Flow

```
Capability Card (Markdown)
    ↓ (startup)
Frontmatter Parser (gray-matter)
    ↓
In-Memory Registry (CapabilityMetadata[])
    ↓ (per-message)
Router Tags + Session Context
    ↓
Selector (relevance scoring)
    ↓
Sanitizer (per-card validation)
    ↓
Injector (real temp file)
    ↓
Bootstrap Context (agent receives)
```

---

## 4. Key Design Decisions

### 4.1 Router-Gated vs Always-On

**Decision:** Router-gated only (zero baseline)

**Rationale:**
- Token-economy goal: Minimize baseline cost
- Most sessions don't need capability awareness
- Router classification already happens (reuse existing work)
- Can force-include via config if needed

**Alternatives considered:**
- ❌ Always inject minimal profile → Conflicts with token-economy
- ❌ Keyword scan on every message → CPU cost, unreliable

**Tradeoff:** Relies on router accuracy (acceptable)

### 4.2 Frontmatter-Only Registry

**Decision:** Single source of truth in markdown frontmatter

**Rationale:**
- No registry duplication (eliminates drift)
- Content updates don't require TS changes
- Standard format (YAML frontmatter)
- Parseable at startup (minimal overhead)

**Alternatives considered:**
- ❌ TS registry + markdown → Two sources of truth (drift risk)
- ❌ JSON registry → Less human-readable, no content co-location

**Tradeoff:** Startup parsing cost (acceptable, <100ms)

### 4.3 Real Temp File vs Synthetic Path

**Decision:** Write real file to `.openclaw_runtime/CAPABILITIES.md`

**Rationale:**
- Guaranteed to work (no synthetic path assumptions)
- Debuggable (can inspect file on disk)
- Standard file I/O (no special handling)

**Alternatives considered:**
- ❌ Synthetic `<synthetic>/CAPABILITIES.md` → Silent failure risk
- ❌ Virtual file scheme → Requires pipeline changes

**Tradeoff:** Disk I/O overhead (negligible, <10ms)

### 4.4 Repo-Only vs Workspace Loading

**Decision:** Repo-only by default, workspace opt-in

**Rationale:**
- Security: Workspace is user-writable (injection surface)
- Capabilities are fork-specific (not user-specific)
- Versioned with code (git-controlled)

**Alternatives considered:**
- ❌ Workspace-first → Security risk
- ❌ Both with equal priority → Confusing precedence

**Tradeoff:** Dev/testing requires opt-in (acceptable)

### 4.5 Per-Card Sanitization

**Decision:** Non-blocking per-card validation

**Rationale:**
- One bad card shouldn't break all injection
- Fail-open is safer (graceful degradation)
- Security boundary is file access, not content parsing

**Alternatives considered:**
- ❌ Global sanitization (throw on error) → Breaks all cards
- ❌ No sanitization → Prompt injection risk

**Tradeoff:** Suspicious cards logged but not blocked

### 4.6 Content-Length Token Caps

**Decision:** Enforce via content length, not estimates

**Rationale:**
- Estimates drift over time (unreliable)
- Content length is accurate proxy (chars / 4 ≈ tokens)
- Hard caps prevent budget overruns

**Alternatives considered:**
- ❌ Metadata estimates only → No enforcement
- ❌ Real tokenizer → Too slow, library dependency

**Tradeoff:** Slight inaccuracy vs speed (acceptable)

### 4.7 Discovery Tool Complementary

**Decision:** Keep discovery tool alongside injection

**Rationale:**
- Debugging: "What capabilities should I have?"
- Status checks: "Is X enabled?"
- User transparency: "What's different about this fork?"

**Alternatives considered:**
- ❌ Injection only → No debugging path
- ❌ Discovery only → Unreliable (agent must remember to call)

**Tradeoff:** Extra tool (worth it for debugging)

---

## 5. Security Model

### 5.1 Threat Model

**Threats:**
1. **Prompt injection via capability cards**
   - Attacker modifies card to inject instructions
   - Mitigation: Repo-only file loading, path validation

2. **Directory traversal**
   - Attacker crafts path like `../../etc/passwd`
   - Mitigation: Realpath validation, prefix check

3. **Symlink attacks**
   - Attacker symlinks card to sensitive file
   - Mitigation: Lstat check, reject symlinks

4. **Content injection**
   - Card contains role confusion markers
   - Mitigation: Sanitization (XML tags, role markers)

### 5.2 Security Controls

**File Access:**
```typescript
// Default: Repo-only
const capabilitiesDir = path.join(repoRoot, 'docs/capabilities');
const realPath = await fs.realpath(filePath);

// Validation
if (!realPath.startsWith(capabilitiesDir)) {
  throw new Error('Path outside allowed directory');
}

const stat = await fs.lstat(filePath);
if (stat.isSymbolicLink()) {
  throw new Error('Symlinks not allowed');
}

if (stat.size > 50 * 1024) {
  throw new Error('File too large');
}
```

**Content Sanitization:**
```typescript
// Per-card validation (non-blocking)
const suspiciousPatterns = [
  /ignore (previous|prior|all) (instructions|prompts)/i,
  /disregard (everything|all|previous)/i,
  /system:\s*you are now/i,
];

for (const pattern of suspiciousPatterns) {
  if (pattern.test(content)) {
    warn(`Skipping ${capId}: suspicious pattern`);
    continue;  // Skip card, not all cards
  }
}

// Sanitize
content = content.replace(/<\/?(?:user|assistant|system)>/gi, '');
content = content.replace(/^(User|Assistant|System):/gim, '[$1]:');
```

**Audit Logging:**
```jsonl
{"timestamp":1707820800,"sessionKey":"main","routerTags":["config"],"injected":[{"id":"token-economy","relevanceScore":1.0,"tokensUsed":580}],"totalTokens":580}
```

### 5.3 Attack Surface

**Minimal:**
- File read (validated paths only)
- Content injection (sanitized, not executed)
- No network access
- No eval/exec

**Trust boundary:**
- Repo-controlled files: Trusted
- Workspace files: Untrusted (opt-in only)

---

## 6. Token Economics

### 6.1 Cost Model

**Baseline (router-gated):**
```
No router tags → No injection → 0 tokens
```

**Dynamic (when triggered):**
```
Router tags match → Inject N capabilities → +200-600 tokens
```

**Frequency estimate:**
```
~5-10% of sessions → ~15-30 sessions/month
30 sessions × 400 tokens avg = 12,000 tokens/month
12,000 tokens × $0.00030/1K (Sonnet input) = $3.60/month
```

**Net impact:**
```
Benefit: Better routing → fewer retries → -2-5 retries/week
Retry cost: ~2000 tokens × 5 = 10,000 tokens/week saved
Monthly: 40,000 tokens saved × $0.00030 = $12/month saved

Net: $12 saved - $3.60 cost = +$8.40/month benefit
```

### 6.2 Kill Switch

**Purpose:** Prevent runaway token costs

**Trigger:** >30% injection rate for 24 hours (min 50 samples)

**Action:** Auto-disable + alert

**Calculation:**
```typescript
const windowMs = 24 * 60 * 60 * 1000;
const cutoff = Date.now() - windowMs;

const recent = metrics.filter(m => m.timestamp >= cutoff);
const triggered = recent.filter(m => m.totalTokens > 0).length;
const rate = triggered / recent.length;

if (rate > 0.30 && recent.length >= 50) {
  // Disable capabilities
  warn(`Auto-disabled: ${triggered}/${recent.length} sessions triggered (>30%)`);
}
```

### 6.3 Budget Control

**Hard caps:**
```typescript
// Per-capability
maxChars: 2400  // ~600 tokens

// Total injection
tokenBudget: 1500  // ~6000 chars

// Enforcement
if (tokensUsed + capability.estimatedTokens > budget) {
  warn('Budget exhausted');
  break;
}
```

---

## 7. Alternative Approaches

### 7.1 Capability Discovery Tool (Rejected as Primary)

**Concept:** Agent calls `list_capabilities` tool when needed

**Pros:**
- Zero token cost until requested
- Agent decides what's relevant

**Cons:**
- Requires agent to remember to call (unreliable)
- Adds latency (~1-2s per call)
- More thinking tokens (agent must decide)

**Verdict:** ✅ **Complementary** (not primary)

Use for:
- Debugging ("What capabilities exist?")
- Status checks ("Is X enabled?")

### 7.2 Always-On Minimal Profile (Rejected)

**Concept:** Always inject 400-token minimal profile

**Pros:**
- Predictable cost
- No heuristics needed

**Cons:**
- Conflicts with token-economy goals
- Wastes tokens on sessions that don't need it
- Baseline cost = bad

**Verdict:** ❌ **Rejected** (token-economy conflict)

### 7.3 Config-Driven Profiles (Hybrid Approach)

**Concept:** Pre-define capability profiles per agent/session

**Pros:**
- Explicit control
- Faster (no keyword scanning)

**Cons:**
- Less dynamic
- Manual configuration
- Higher baseline cost

**Verdict:** ✅ **Future enhancement** (not v1)

**Hybrid approach (future):**
```json
{
  "agents": {
    "main": {
      "capabilityProfile": "minimal",
      "forceInclude": ["token-economy"]
    }
  }
}
```

### 7.4 Runtime Tool Registry (Complementary)

**Concept:** Expose fork-specific tools that reveal capabilities

**Pros:**
- Self-documenting through use
- Zero prompt tokens

**Cons:**
- Agent must discover organically (slow)
- Doesn't help with "why is this different?"

**Verdict:** ✅ **Complementary**

Examples:
- `token_economy_status` - Show routing decisions
- `memory_search` - Reveal memory system

---

## 8. Tradeoffs

### 8.1 Router Dependency

**Tradeoff:** Relies on router accuracy

**Risk:** Router misses relevant tags → no injection

**Mitigation:**
- Keyword fallback (current message scan)
- Force-include config option
- Adjust router training over time

**Acceptable:** Router is core OpenClaw component (stable)

### 8.2 Startup Parsing Cost

**Tradeoff:** Registry built at startup (parse frontmatter)

**Cost:** ~50-100ms for 5-10 capability cards

**Mitigation:**
- In-memory cache after first parse
- Lazy loading (only parse when first needed)

**Acceptable:** One-time cost, negligible

### 8.3 Disk I/O for Injection

**Tradeoff:** Write real file on every injection

**Cost:** ~10ms write + ~5ms read

**Mitigation:**
- Only triggered 5-10% of sessions
- Async I/O (non-blocking)

**Acceptable:** Minimal latency, worth reliability

### 8.4 Content Updates Require Restart

**Tradeoff:** Capability card updates need gateway restart

**Risk:** Stale content if update without restart

**Mitigation:**
- Clear cache on config reload
- Hot-reload capability cards (future)

**Acceptable:** Content updates are rare (~weekly)

### 8.5 False Positives (Over-Injection)

**Tradeoff:** Router might trigger when not needed

**Risk:** Wasted tokens on unnecessary injection

**Mitigation:**
- Confidence thresholds per capability
- Exclude config option
- Metrics monitoring (kill switch)

**Acceptable:** Better over-inform than under-inform

---

## 9. Extension Points

### 9.1 Custom Selectors

**Hook:** `selectRelevantCapabilities`

**Use case:** Alternative selection logic (ML-based, LLM-based)

**Example:**
```typescript
// ML-based selector (future)
const embeddings = await embedCapabilities(registry);
const queryEmbedding = await embedMessage(context.currentMessage);
const scores = cosineSimilarity(queryEmbedding, embeddings);
return topK(scores, 3);
```

### 9.2 Custom Sanitizers

**Hook:** `validateAndSanitizeContent`

**Use case:** Domain-specific validation rules

**Example:**
```typescript
// Security domain: Extra checks
if (capability.id.startsWith('security-')) {
  // Enforce stricter patterns
  // Require digital signature
  // Validate against schema
}
```

### 9.3 Alternative Metrics Backends

**Hook:** `logCapabilityMetrics`

**Use case:** Send metrics to external system (Prometheus, Datadog)

**Example:**
```typescript
// Datadog integration
statsd.increment('capabilities.injected', 1, {
  tags: [`capability:${cap.id}`, `session:${sessionKey}`]
});
```

### 9.4 Dynamic Capability Loading

**Hook:** `getCapabilityRegistry`

**Use case:** Load capabilities from external source (API, database)

**Example:**
```typescript
// Load from API (future)
const response = await fetch('https://capability-registry.example.com/api/capabilities');
const capabilities = await response.json();
return capabilities.map(parseCapabilityMetadata);
```

---

## 10. Future Considerations

### 10.1 Hot-Reload Capability Cards

**Goal:** Update cards without gateway restart

**Approach:**
- Watch `docs/capabilities/` directory
- Re-parse on file change
- Clear registry cache
- Log reload event

**Benefit:** Faster iteration during development

### 10.2 LLM-Based Selection

**Goal:** Use LLM to decide which capabilities are relevant

**Approach:**
```
User message + Capability descriptions → LLM → Relevance scores
```

**Benefit:** More accurate selection (context-aware)

**Cost:** Extra LLM call (~$0.01 per selection)

**Trade off:** Accuracy vs latency vs cost

### 10.3 User-Editable Capabilities

**Goal:** Allow users to define custom capabilities

**Requirements:**
- Strict sandboxing (isolated from system capabilities)
- Content validation (schema enforcement)
- Separate directory (`~/.openclaw/user-capabilities/`)
- Explicit enable (`allowUserCapabilities: true`)

**Risk:** Prompt injection surface (high scrutiny needed)

### 10.4 Capability Marketplace

**Goal:** Share capabilities across OpenClaw forks

**Approach:**
- Central registry (clawhub.com)
- Download + install capabilities
- Version management
- Trust model (signatures, reviews)

**Benefit:** Community-driven capability ecosystem

### 10.5 Multi-Tier Injection

**Goal:** Inject capabilities at multiple points (bootstrap, mid-conversation, post-tool-use)

**Example:**
```
Bootstrap: Core capabilities (token-economy)
Mid-conversation: Context-specific (memory-system when user asks "remember")
Post-tool-use: Tool-triggered (security when file_write called)
```

**Benefit:** More granular, just-in-time-er injection

**Complexity:** Higher orchestration overhead

---

## Conclusion

**Architecture Status:** ✅ **Approved for Implementation**

**Key Strengths:**
- Zero baseline token cost (router-gated)
- Single source of truth (frontmatter-only)
- Security-first (repo-only, validated paths)
- Maintainable (content-only updates)
- Safe rollback (config + code + git)

**Known Limitations:**
- Relies on router accuracy (mitigated with fallbacks)
- Startup parsing cost (acceptable, <100ms)
- Content updates require restart (acceptable, rare)

**Recommendation:** Proceed with implementation per `REFINED_IMPLEMENTATION_PLAN.md`

**Next Review:** After Phase 2 (selection & injection logic) to validate assumptions

---

**Document Version:** 2.0  
**Last Updated:** 2026-02-13  
**Status:** Living document (update as implementation progresses)
