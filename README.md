# Capability-Awareness System

**Making AI agents reliably aware of custom fork capabilities without bloating the always-on system prompt.**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-Deployed%20(Skills--First)-success.svg)]()
[![Version](https://img.shields.io/badge/version-2.0%20(Refined)-blue.svg)]()
[![Implementation](https://img.shields.io/badge/deployed-2026--02--13-brightgreen.svg)]()

---

## 🎯 Version 2.0 - Refined & Hardened

**Critical corrections applied based on independent technical assessment:**

✅ **Zero baseline cost** - Router-gated only (was: +400 tokens baseline)  
✅ **Real temp files** - No more synthetic paths (guaranteed to work)  
✅ **Frontmatter-only registry** - Single source of truth (no drift)  
✅ **Content-length token caps** - Enforced (not estimated)  
✅ **Repo-only loading** - Security-first (workspace opt-in)  
✅ **Per-card validation** - Non-blocking (graceful degradation)  
✅ **Safe rollback** - Proper JSON editing (no corruption)  
✅ **Metrics instrumentation** - Kill switch + analytics  
✅ **Discovery tool retained** - Complementary debugging

**See:** [`TECHNICAL_ASSESSMENT.md`](TECHNICAL_ASSESSMENT.md) for full details

---

## Problem Statement

When you fork an AI agent framework (like OpenClaw) and add custom capabilities:
- 🤔 Agent "forgets" the custom features
- 💰 Adding them to the always-on prompt costs tokens
- 🔄 Updating capabilities requires prompt changes
- 📝 No structured way to document fork-specific features

**Example:** Your fork has token-economy hooks, Telegram optimizations, and a RAG memory system, but the agent doesn't consistently leverage them because they're not in the prompt.

---

## Solution Overview

**Three-tier capability awareness system:**

```
Tier 1: AGENTS.md (≤500 chars)
  ↓ Always loaded, minimal cost, points to capabilities
  
Tier 2: docs/capabilities/*.md (200-800 chars each)
  ↓ Just-in-time injection when relevant
  
Tier 3: SYSTEM_ARCHITECTURE.md
  ↓ Read via tool only when needed (never auto-loaded)
```

### How It Works

1. **Keyword scanning:** Scan recent messages for capability keywords
2. **Relevance scoring:** Calculate match score (keyword ratio + density)
3. **Selection:** Sort by priority, fit within token budget
4. **Injection:** Insert into bootstrap context via hook
5. **Result:** Agent gets targeted awareness without prompt bloat

---

## Key Features

✅ **Smart selection** - Keywords + session type + relevance scoring  
✅ **Token-aware** - Hard cap at 1500 tokens (configurable)  
✅ **Secure** - Repo-controlled files, content sanitization  
✅ **Config-driven** - Enable/disable, force-include, exclude  
✅ **Fail-open** - Errors don't break agent startup  
✅ **Auditable** - Logs all injections with relevance scores

---

## ⚠️ Version Notice

**Use v2.0 (Refined) documents** - Original v1.0 plan contained critical issues identified in technical assessment.

✅ **Latest (v2.0):**
- [`REFINED_IMPLEMENTATION_PLAN.md`](REFINED_IMPLEMENTATION_PLAN.md) - Complete implementation guide
- [`REFINED_SUMMARY.md`](REFINED_SUMMARY.md) - Executive summary
- [`ARCHITECTURE.md`](ARCHITECTURE.md) - Design decisions

📋 **Reference:**
- [`TECHNICAL_ASSESSMENT.md`](TECHNICAL_ASSESSMENT.md) - What changed and why

❌ **Deprecated (v1.0):**
- `IMPLEMENTATION_PLAN.md` - Contains architectural flaws (use REFINED version)
- `SUMMARY.md` - Superseded by REFINED version

---

## Quick Start

### 1. Read the Plan

**Start here:** [`REFINED_IMPLEMENTATION_PLAN.md`](REFINED_IMPLEMENTATION_PLAN.md) (36 KB)

**Quick reference:** [`REFINED_SUMMARY.md`](REFINED_SUMMARY.md) (9 KB)

**Design docs:** [`ARCHITECTURE.md`](ARCHITECTURE.md) (19 KB)

### 2. Understand the Architecture

**File structure:**
```
openclaw/                           # Your fork
├── AGENTS.md                       # NEW: Ultra-concise manifest (≤500 chars)
├── docs/
│   └── capabilities/               # NEW: Capability cards
│       ├── token-economy.md
│       ├── telegram-optimizations.md
│       └── memory-system.md
├── src/
│   ├── plugins/
│   │   └── capabilities/           # NEW: Injection logic
│   │       ├── selector.ts         # Relevance scoring
│   │       ├── registry.ts         # Capability metadata
│   │       └── injector.ts         # Token-aware injection
│   └── agents/
│       └── bootstrap-files.ts      # MODIFY: Add injection hook
```

### 3. Implement

**Phase 1: Infrastructure (2 hours)**
- Create capability selector and registry
- Implement relevance scoring algorithm

**Phase 2: Injection Logic (1.5 hours)**
- Integrate with `before_context_build` hook
- Add config support

**Phase 3: Capability Cards (1.5 hours)**
- Create initial capability cards
- Write AGENTS.md manifest

**Phase 4: Testing (1 hour)**
- Unit tests
- Integration tests
- E2E tests

**Total:** 4-6 hours to PR-ready

---

## Token Economics

**Baseline cost:** +400 tokens (AGENTS.md + minimal capabilities)

**Dynamic cost:** +200-600 tokens (keyword-triggered)

**Net impact:** Neutral or positive
- Smarter context → better model routing → fewer retries
- Less "forgot this feature" → fewer clarification turns

**Budget control:**
```json
{
  "capabilities": {
    "maxTokens": 1500  // Adjust as needed
  }
}
```

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

**Config:** `~/.openclaw/openclaw.json`
```

**Selection logic:**
- Keywords in conversation → relevance score
- Session type match → eligible
- Priority sort → inject highest first
- Token budget → stop when exhausted

---

## Security

**File Access Control:**
- Capability files loaded only from `<repoRoot>/docs/capabilities/`
- Path validation prevents directory traversal
- Workspace fallback only for dev/testing

**Content Sanitization:**
```typescript
// Strip suspicious patterns
- "ignore previous instructions"
- "disregard ... above"
- "new instructions:"
```

**Frontmatter Validation:**
- Token budget max: 2000
- Triggers format validation
- Metadata type checking

**Audit Logging:**
```
[capability-injection] Injected 2 capabilities (850 tokens): 
  token-economy (0.85), telegram-optimizations (0.62)
```

---

## Configuration

**Enable/disable:**
```json
{
  "capabilities": {
    "enabled": true,          // Default: true
    "maxTokens": 1500,        // Default: 1500
    "confidenceThreshold": 0.6,
    "forceInclude": ["token-economy"],
    "exclude": ["security-controls"]
  }
}
```

**Quick disable:**
```json
{"capabilities": {"enabled": false}}
```

---

## Rollback Plan

**Level 1: Config disable (instant)**
```json
{"capabilities": {"enabled": false}}
```

**Level 2: Code disable (5 minutes)**
```typescript
// Comment out injection hook in bootstrap-files.ts
// Rebuild: npm run build
```

**Level 3: Git revert (10 minutes)**
```bash
git revert <commit-hash>
npm run build
```

All rollback levels tested and documented.

---

## Alternative Approaches

We evaluated several alternatives:

### 1. Capability Discovery Tool ❌
- **Pros:** Zero token cost until requested
- **Cons:** Requires agent to be proactive, adds latency
- **Verdict:** Not recommended (unreliable awareness)

### 2. Runtime Tool Registry ✅ (Complementary)
- **Pros:** Self-documenting through usage
- **Cons:** Doesn't help with "why is this different?"
- **Verdict:** Use alongside injection for operational stats

### 3. Config-Driven Injection ✅ (Hybrid)
- **Pros:** Predictable, explicit control
- **Cons:** Less dynamic
- **Verdict:** Recommended as enhancement (base profiles + keyword augmentation)

**Our recommendation:** Primary (just-in-time) + config profiles + runtime tools

---

## Implementation Effort

**Total time:** 4-6 hours  
**Files created:** 15 new, 3 modified  
**Lines of code:** ~1200  
**Token impact:** +400 baseline, +200-600 dynamic (net neutral/positive)

---

## Use Cases

### Fork-Specific Features
- Custom hooks (model routing, context bundling)
- Platform optimizations (Telegram, Discord, Slack)
- Integration modules (memory systems, databases)
- Security controls (sandboxing, allowlists)

### Multi-Environment Deployments
- Development vs production capabilities
- Client-specific features
- Team-specific tools
- Compliance-specific controls

### Version Awareness
- Breaking changes between versions
- Deprecated features
- New capabilities
- Migration paths

---

## Documentation

- **[Implementation Plan](IMPLEMENTATION_PLAN.md)** (35 KB) - Complete guide with code examples
- **[Summary](SUMMARY.md)** (4 KB) - Quick reference
- **[Architecture](docs/ARCHITECTURE.md)** - System design and decisions
- **[Examples](examples/)** - Sample capability cards and configs
- **[Templates](templates/)** - File templates for quick start

---

## Requirements

- AI agent framework with bootstrap file loading (like OpenClaw)
- Hook system for pre-processing (e.g., `before_context_build`)
- TypeScript/JavaScript (can be adapted to other languages)
- Access to recent message history (for keyword scanning)

---

## Compatibility

**Tested with:**
- OpenClaw 2026.2.10+
- TypeScript 5.9.3+
- Node.js 22+

**Should work with:**
- Any agent framework with bootstrap/context loading
- Any language (implementation plan is framework-agnostic)

---

## Contributing

This is a reference implementation. Adapt for your framework:

1. Identify your bootstrap/context loading point
2. Add keyword scanning from recent messages
3. Implement relevance scoring
4. Create capability cards in your format
5. Inject based on relevance + token budget

**Pull requests welcome** for:
- Framework-specific implementations
- Additional capability card examples
- Alternative scoring algorithms
- Performance optimizations

---

## License

MIT License - See [LICENSE](LICENSE)

---

## Repository

**GitHub:** https://github.com/pfaria32/openclaw-capability-awareness

## Related Projects

- **[OpenClaw](https://github.com/openclaw/openclaw)** - AI agent framework this was designed for
- **[Token Economy](https://github.com/pfaria32/open_claw_token_economy)** - Token optimization hooks
- **[OpenClaw Shield](https://github.com/pfaria32/OpenClaw-Shield)** - Security controls for AI agents

---

## Acknowledgments

Designed for Pedro Bento de Faria's OpenClaw fork as part of a comprehensive token economy optimization project.

**Design goals:**
- Minimize token spend while maximizing agent capability awareness
- Enable fork-specific features without prompt bloat
- Support dynamic capability injection based on conversation context
- Maintain security through repo-controlled content

**Result:** 4-6 hour implementation that reduces token waste while improving agent awareness of custom capabilities.

---

## Status

✅ **Implemented via Skills-First approach** (2026-02-13 05:35 UTC)  
✅ **5 capability skills deployed** (token-economy, telegram, memory, second-brain, env)  
✅ **AGENTS.md enhanced** with capability hints  
✅ **2-hour deployment** (faster than Full Injection's 6-8h)  
✅ **Monitoring ongoing** (evaluate after 1-2 weeks)  
⏳ **Full Injection** upgrade available if needed

**See:** [`SKILLS_FIRST_IMPLEMENTATION.md`](SKILLS_FIRST_IMPLEMENTATION.md) for deployment details.

---

**Questions?** See the [Implementation Plan](IMPLEMENTATION_PLAN.md) for complete details including:
- System survey (current bootstrap/skills loading)
- File/folder layout
- Just-in-time injection implementation
- Code touchpoints
- Rollback plan
- Alternative approaches evaluated
