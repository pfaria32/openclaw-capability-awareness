# Capability Awareness System - Technical Assessment Report

**Date:** 2026-02-13  
**Reviewer:** Independent Architectural Review  
**Target:** OpenClaw Fork (Token-Economy Instance)  
**Document Version:** 1.0

---

## Executive Summary

The proposed Capability Awareness System is **architecturally sound in intent** but contains **critical implementation flaws** that must be corrected before deployment.

**Verdict:** ✅ **Approve conceptually** | ❌ **Block implementation until corrections applied**

---

## Assessment Overview

### What Is Strong (Preserve)

✅ Three-tier awareness model (AGENTS.md → capability cards → deep docs)  
✅ `before_context_build` hook location  
✅ Fail-open behavior on errors  
✅ Config kill switch  
✅ Hard token budget concept

### What Must Change (Critical)

⚠️ **8 architectural issues** identified  
⚠️ **3 security vulnerabilities** found  
⚠️ **2 token-economy regressions** detected  
⚠️ **1 unsafe rollback procedure** documented

---

## Critical Issues (Blocking)

### 1. Synthetic File Injection Risk ⛔ (High Severity)

**Problem:**
```typescript
path: "<synthetic>/CAPABILITIES.md"
content: capabilityContent
```

Unless the embedder explicitly consumes `.content` and ignores `path`, this will **silently fail**.

**Required Fix (Choose One):**

**Option A: Real Temp File (Recommended)**
```typescript
const runtimeDir = path.join(workspaceDir, '.openclaw_runtime');
await fs.mkdir(runtimeDir, { recursive: true });

const capFilePath = path.join(runtimeDir, 'CAPABILITIES.md');
await fs.writeFile(capFilePath, capabilityContent, 'utf-8');

const capabilityFile: WorkspaceBootstrapFile = {
  name: "CAPABILITIES.md" as any,
  path: capFilePath,  // Real path
  content: capabilityContent,
  missing: false,
};
```

**Option B: Explicit Virtual File Support (Higher Effort)**
```typescript
// Add to bootstrap pipeline
if (file.path.startsWith('virtual://')) {
  // Use .content, ignore path
  return file.content;
}
```

**Decision required:** Option A (safest) or implement Option B with full pipeline support.

---

### 2. Registry Duplication (Drift Risk) ⛔

**Problem:**
- Markdown frontmatter: `capability_id`, `triggers`, `priority`, `token_budget`
- TS registry: `CAPABILITY_REGISTRY` with same data

**Two sources of truth = guaranteed drift.**

**Required Fix:**

**Use frontmatter as single source:**
```typescript
// At startup
async function buildRegistry(capabilitiesDir: string): Promise<CapabilityMetadata[]> {
  const files = await fs.readdir(capabilitiesDir);
  const registry: CapabilityMetadata[] = [];

  for (const file of files) {
    if (!file.endsWith('.md')) continue;

    const content = await fs.readFile(path.join(capabilitiesDir, file), 'utf-8');
    const { data } = matter(content);  // Parse frontmatter

    registry.push({
      id: data.capability_id,
      name: data.name || data.capability_id,
      description: data.description || '',
      filePath: `docs/capabilities/${file}`,
      triggers: data.triggers,
      priority: data.priority,
      estimatedTokens: data.token_budget,
    });
  }

  return registry;
}

// Cache in memory
let CAPABILITY_REGISTRY: CapabilityMetadata[] | null = null;
```

**No manual registry file.**

---

### 3. getRecentMessages Is Hypothetical ⛔

**Problem:**
```typescript
const recentMessages = await getRecentMessages(params.sessionKey, 5);
```

**This function does not exist in the codebase.**

**Required Fix (Options):**

**Option 1: Current message only**
```typescript
// Use router classification tags instead
const context: SelectionContext = {
  currentMessage: params.currentMessage,
  sessionType: isSubagentSessionKey(params.sessionKey) ? "isolated" : "main",
  // ...
};
```

**Option 2: Router tags (Preferred)**
```typescript
// If router provides tags like: routing:cost, routing:integration
const routerTags = params.routerClassification?.tags || [];
const relevantCapabilities = CAPABILITY_REGISTRY.filter(cap =>
  routerTags.some(tag => cap.triggers.keywords.includes(tag))
);
```

**Option 3: Implement Real History Accessor**
```typescript
// Would require access to session storage
async function getRecentMessages(sessionKey: string, limit: number): Promise<string[]> {
  // Implementation depends on session storage architecture
  // May not be worth the complexity
}
```

**Recommendation:** Use Option 2 (router tags) + current message text.

---

### 4. Token Budget Is Not Real ⛔

**Problem:**
```typescript
estimatedTokens: 600  // Just metadata, not enforced
```

**This will drift and is unreliable.**

**Required Fix:**

**Enforce real caps:**
```typescript
interface CapabilityMetadata {
  // ...
  maxChars: number;         // Hard char cap
  estimatedTokens?: number; // Keep for logging only
}

function enforceTokenBudget(content: string, maxTokens: number): string {
  const approxTokens = Math.ceil(content.length / 4);  // Rough estimate
  
  if (approxTokens > maxTokens) {
    const targetChars = maxTokens * 4;
    return content.slice(0, targetChars) + '\n\n[Content truncated to fit token budget]';
  }
  
  return content;
}
```

**Optional:** Use real tokenizer if available (tiktoken for GPT, claude-tokenizer for Anthropic).

---

### 5. Token-Economy Alignment Conflict ⛔

**Problem:**
```
Baseline: +400 tokens (AGENTS.md + minimal profile)
```

**This directly conflicts with token-economy goals. You will pay +400 tokens on EVERY message.**

**Required Change:**

**Replace "always inject minimal" with "inject only when router requests":**

```typescript
// Only inject if router tags request capabilities
const routerTags = params.routerClassification?.tags || [];
const needsCapabilities = routerTags.some(tag => 
  ['config', 'integration', 'deployment', 'memory', 'token-economy', 'debug'].includes(tag)
);

if (!needsCapabilities && !capabilityConfig.forceInclude?.length) {
  return { files: bootstrapFiles, result: { injectedCapabilities: [], totalTokensUsed: 0, truncated: false } };
}
```

**Baseline injection must be 0 tokens. Only inject on-demand.**

---

## Security Issues (High Priority)

### 6. Workspace Fallback Is Dangerous ⛔

**Problem:**
```typescript
const candidates = [
  path.join(workspaceDir, relativePath),  // User-writable
  path.join(repoRoot, relativePath)
];
```

**Workspace is user-writable = injection surface.**

**Required Change:**

**Default: repo-only**
```typescript
capabilities: {
  allowWorkspaceCards: false  // Default: false
}
```

**If enabled: strict validation**
```typescript
if (capabilityConfig.allowWorkspaceCards) {
  const realPath = await fs.realpath(candidate);
  
  // Must be within workspace
  if (!realPath.startsWith(workspaceDir)) {
    throw new Error('Path escapes workspace');
  }
  
  // No symlinks
  const stat = await fs.lstat(candidate);
  if (stat.isSymbolicLink()) {
    throw new Error('Symlinks not allowed');
  }
  
  // Size cap
  if (stat.size > 50 * 1024) {  // 50 KB
    throw new Error('File too large');
  }
}
```

---

### 7. Path Validation Must Be Explicit ⛔

**Required checks:**
```typescript
async function validateCapabilityPath(filePath: string, repoRoot: string): Promise<void> {
  // Resolve to real path (follows symlinks)
  const realPath = await fs.realpath(filePath);
  
  // Must be within repo capabilities dir
  const capabilitiesDir = path.join(repoRoot, 'docs/capabilities');
  if (!realPath.startsWith(capabilitiesDir)) {
    throw new Error(`Capability file outside allowed directory: ${realPath}`);
  }
  
  // No symlinks
  const stat = await fs.lstat(filePath);
  if (stat.isSymbolicLink()) {
    throw new Error(`Symlinks not allowed: ${filePath}`);
  }
  
  // Size cap
  if (stat.size > 50 * 1024) {
    throw new Error(`File too large: ${stat.size} bytes`);
  }
}
```

---

### 8. Sanitization Is Weak ⛔

**Problem:**
```typescript
const suspiciousPatterns = [
  /ignore previous instructions/i,
  /disregard.*above/i,
  /new instructions:/i,
];

// Throws on match (breaks all injection)
```

**Issues:**
- Phrase-based (easy to bypass)
- Treats as security boundary (it's not)
- Throws on error (breaks all cards)

**Required Changes:**

**Per-card validation:**
```typescript
function validateCapabilityContent(capId: string, content: string): {
  valid: boolean;
  reason?: string;
} {
  const suspiciousPatterns = [
    /ignore (previous|prior|all|above) (instructions|prompts|context)/i,
    /disregard (everything|all|previous|prior) (above|instructions)/i,
    /new instructions:/i,
    /system:\s*you are now/i,
  ];

  for (const pattern of suspiciousPatterns) {
    if (pattern.test(content)) {
      return { 
        valid: false, 
        reason: `Suspicious pattern detected: ${pattern}` 
      };
    }
  }

  return { valid: true };
}

// Usage
const validation = validateCapabilityContent(cap.metadata.id, content);
if (!validation.valid) {
  warn(`[capability] Skipping ${cap.metadata.id}: ${validation.reason}`);
  continue;  // Skip this card, not all cards
}
```

**Real security boundary:**
- Trusted source only (repo-controlled)
- Strict path validation
- Size limits
- Audit logging

---

## Content Issues (Medium Priority)

### 9. Drift-Prone Hard Numbers ⚠️

**Problem in example cards:**
```markdown
- 57 files indexed
- 76 chunks embedded
- $60–105/month savings
```

**These will become false over time.**

**Required Changes:**
```markdown
# Before
- 57 files indexed, 76 chunks embedded

# After
- Run `qmd status` for current counts
```

```markdown
# Before
- $60–105/month savings

# After
- See `~/.openclaw/token-audit/*.jsonl` for current savings
```

**Guideline:** Reference live data sources, not snapshots.

---

### 10. Hard-coded Credentials ⚠️

**Problem:**
```markdown
Account: user@example.com
```

**Should not be in generic capability cards.**

**Required Change:**
```markdown
# Before
Account: user@example.com

# After
Account: Configured via env (check `openclaw config` for details)
```

**Or:** Use environment catalog tool.

---

### 11. Config Key Confusion ⚠️

**Problem:**
```markdown
Set `webchat.capabilities.inlineButtons`
```

**Verify:** Is this the correct key? Should it be channel-scoped?

**Action:** Confirm with config schema or rename to `channels.telegram.capabilities.inlineButtons`.

---

## Unsafe Procedures (Critical)

### 12. Rollback JSON Append Is Unsafe ⛔

**Problem:**
```bash
cat >> ~/.openclaw/openclaw.json << 'EOF'
{
  "capabilities": { "enabled": false }
}
EOF
```

**This will corrupt JSON (appends to end, not merge).**

**Required Replacement:**

**Option A: jq (Safe)**
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

jq '.capabilities.enabled = false' ~/.openclaw/openclaw.json > /tmp/openclaw.json.tmp
mv /tmp/openclaw.json.tmp ~/.openclaw/openclaw.json

# Restart
docker compose restart openclaw-gateway
```

**Option B: Node Script (Safer)**
```javascript
// disable-capabilities.js
const fs = require('fs');
const config = JSON.parse(fs.readFileSync(process.env.HOME + '/.openclaw/openclaw.json', 'utf-8'));
config.capabilities = config.capabilities || {};
config.capabilities.enabled = false;
fs.writeFileSync(process.env.HOME + '/.openclaw/openclaw.json', JSON.stringify(config, null, 2));
```

```bash
node disable-capabilities.js
docker compose restart openclaw-gateway
```

---

## Rejected Alternative: Discovery Tool

**Problem:**

Plan says: "Capability Discovery Tool → ❌ Not recommended"

**This is incorrect.**

**Discovery tool should be:**
- Complementary (not replacement)
- Used for debugging
- Used for env presence checks
- Referenced in AGENTS.md

**Example tool:**
```typescript
// list_capabilities tool
{
  name: "list_capabilities",
  description: "List available fork-specific capabilities and check their status",
  parameters: {
    type: "object",
    properties: {
      detail: { type: "string", enum: ["summary", "full"] }
    }
  }
}
```

**Use cases:**
- Agent debugging: "What capabilities am I supposed to have?"
- User debugging: "Is token-economy actually enabled?"
- Status checks: "Show me which capabilities are active"

**Recommendation:** Keep discovery tool as complementary, not primary.

---

## Maintenance Requirements

**Before deployment, add:**

### Metrics Logging

```typescript
interface CapabilityMetrics {
  sessionId: string;
  timestamp: number;
  injected: Array<{ id: string; relevanceScore: number; tokensUsed: number }>;
  totalTokens: number;
  selectionTimeMs: number;
}

// Log to: ~/.openclaw/capability-metrics.jsonl
```

**Track:**
- % sessions where injection triggered
- Avg tokens injected per session
- Most frequently injected capabilities
- Model routing before/after injection
- Retry/tool-failure rate correlation

### Kill Switch

**If injection triggers > 30% of sessions for 24h:**
```typescript
capabilities: {
  autoDisableThreshold: 0.30,  // 30% of sessions
  autoDisableWindowHours: 24
}
```

**Action:** Auto-disable or alert (don't silently burn tokens).

---

## Recommended Final Architecture

### For Your OpenClaw Fork

**Always-On (Tier 1):**
- Tight AGENTS.md (≤600 tokens)
- No automatic injection

**On-Demand Injection (Tier 2):**
- Triggered by router classification tags
- Real file written under `.openclaw_runtime/`
- Strict path validation
- Repo-only loading (default)
- Hard token caps enforced

**Complementary Tools (Tier 3):**
- `list_capabilities` - Discovery tool
- `token_economy_status` - Operational stats
- `env_presence_check` - Environment validation

**Single Source of Truth:**
- Markdown frontmatter (parsed at startup)
- No separate TS registry file

---

## Required Changes Before Approval

**Implementation MUST NOT proceed until:**

1. ✅ **Synthetic file injection** → Real temp file or explicit virtual support
2. ✅ **Registry duplication** → Removed (frontmatter only)
3. ✅ **Workspace fallback** → Removed or strictly gated
4. ✅ **Token caps** → Enforced by content length
5. ✅ **Baseline injection** → Removed (router-gated only)
6. ✅ **Rollback JSON append** → Replaced with safe method
7. ✅ **Metrics instrumentation** → Added
8. ✅ **getRecentMessages dependency** → Resolved or removed

---

## Approval Checklist

**Architectural:**
- [ ] Synthetic file → Real file or virtual scheme
- [ ] Registry → Frontmatter only
- [ ] Token budget → Content-length enforced
- [ ] Baseline injection → Removed

**Security:**
- [ ] Workspace fallback → Disabled by default
- [ ] Path validation → Explicit realpath checks
- [ ] Sanitization → Per-card, non-blocking
- [ ] Audit logging → All injections logged

**Token Economy:**
- [ ] Zero baseline cost
- [ ] Router-gated injection only
- [ ] Kill switch implemented
- [ ] Metrics instrumentation added

**Maintenance:**
- [ ] Hard-coded numbers → Dynamic references
- [ ] Credentials → Removed or env-based
- [ ] Rollback procedure → Safe JSON editing

**Testing:**
- [ ] Unit tests for selector
- [ ] Unit tests for injector
- [ ] Integration test for full flow
- [ ] E2E test with real gateway
- [ ] Rollback tested

---

## Final Recommendation

**Direction:** ✅ **Correct and valuable**  
**Execution Plan:** ⚠️ **Needs refinement**

**Approve conceptually, but require corrections before implementation.**

**Estimated effort with corrections:** 6-8 hours (up from 4-6 due to required changes)

**Token impact after corrections:**
- Baseline: **0 tokens** (router-gated)
- Dynamic: +200-600 tokens (when triggered)
- **Net positive** (reduced retries + better routing)

---

**Reviewer Notes:**

This assessment prioritizes:
1. **Safety:** No silent failures
2. **Token economy:** Zero baseline cost
3. **Maintainability:** Single source of truth
4. **Security:** Strict path validation

The system can be excellent with these corrections. Without them, it will either fail silently or cost more tokens than it saves.

---

**Next Step:** Create refined implementation plan incorporating all corrections.
