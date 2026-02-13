# Architecture

**Capability-Awareness System - Technical Architecture**

---

## Overview

The Capability-Awareness System is a **just-in-time content injection framework** that makes AI agents aware of fork-specific capabilities based on conversation context, without bloating the always-on system prompt.

---

## Core Concepts

### 1. Three-Tier Information Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Tier 1: AGENTS.md (≤500 chars)                         │
│ Always loaded • Minimal tokens • Points to capabilities│
├─────────────────────────────────────────────────────────┤
│ Tier 2: Capability Cards (200-800 chars each)          │
│ Just-in-time injection • Context-based • Token-aware   │
├─────────────────────────────────────────────────────────┤
│ Tier 3: SYSTEM_ARCHITECTURE.md                         │
│ Deep reference • Never auto-loaded • Read via tool     │
└─────────────────────────────────────────────────────────┘
```

**Design rationale:**
- **Tier 1:** Baseline awareness (what exists)
- **Tier 2:** Targeted details (when relevant)
- **Tier 3:** Deep knowledge (when requested)

### 2. Just-in-Time Injection Flow

```
┌─────────────────┐
│ User Message    │
└────────┬────────┘
         │
         ↓
┌─────────────────────┐
│ Extract Keywords    │ (Last N messages)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Score Relevance     │ (Keyword match + density)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Filter by Threshold │ (e.g., ≥0.6)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Sort by Priority    │ (High → Low)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Fit in Token Budget │ (Max 1500 tokens)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Inject into Context │ (via hook)
└────────┬────────────┘
         │
         ↓
┌─────────────────────┐
│ Agent Processes     │ (with capability awareness)
└─────────────────────┘
```

---

## Components

### 1. Capability Registry

**Location:** `src/plugins/capabilities/registry.ts`

**Purpose:** Central metadata store for all capabilities

**Structure:**
```typescript
export interface CapabilityMetadata {
  id: string;                    // Unique identifier
  name: string;                  // Human-readable name
  description: string;           // Brief description
  filePath: string;              // Relative path to card
  triggers: {
    keywords: string[];          // Trigger keywords
    sessionTypes: string[];      // Eligible session types
    confidenceThreshold: number; // Minimum relevance score
  };
  priority: number;              // Injection priority (higher = first)
  estimatedTokens: number;       // Token budget for this card
}
```

**Example:**
```typescript
{
  id: "token-economy",
  name: "Token Economy Hooks",
  description: "Model routing, context bundling, zero-token heartbeat",
  filePath: "docs/capabilities/token-economy.md",
  triggers: {
    keywords: ["cost", "tokens", "model", "routing", "budget"],
    sessionTypes: ["main", "isolated"],
    confidenceThreshold: 0.6,
  },
  priority: 10,
  estimatedTokens: 600,
}
```

### 2. Capability Selector

**Location:** `src/plugins/capabilities/selector.ts`

**Purpose:** Determine which capabilities to inject based on context

**Key functions:**

#### `selectRelevantCapabilities(context)`
Selects capabilities based on recent messages and session type.

**Input:**
```typescript
{
  recentMessages: string[],  // Last N messages
  sessionType: string,       // main | isolated
  tokenBudget: number,       // Max tokens to inject
}
```

**Output:**
```typescript
[
  {
    metadata: CapabilityMetadata,
    relevanceScore: number,
    content: string,
  },
  // ...
]
```

#### `calculateRelevanceScore(capability, context)`
Calculates relevance score for a capability.

**Algorithm:**
```typescript
if (no messages) {
  return priority / 100;  // Base score from priority
}

recentText = messages.join(" ").toLowerCase();
matchedKeywords = keywords.filter(k => recentText.includes(k));

if (matchedKeywords.length === 0) {
  return 0;
}

keywordRatio = matched / total;
matchCount = sum of all keyword occurrences;
density = min(matchCount / 100, 1.0);

score = keywordRatio * 0.7 + density * 0.3;
return score;
```

**Scoring factors:**
- **Keyword ratio (70%):** % of keywords matched
- **Keyword density (30%):** Frequency of matches

### 3. Capability Injector

**Location:** `src/plugins/capabilities/injector.ts`

**Purpose:** Inject selected capabilities into bootstrap context

**Key functions:**

#### `injectCapabilities(bootstrapFiles, context)`
Adds capability cards to bootstrap files.

**Process:**
1. Select relevant capabilities
2. Create synthetic CAPABILITIES.md file
3. Insert after AGENTS.md in bootstrap files
4. Return augmented file list

**Output:**
```typescript
{
  files: WorkspaceBootstrapFile[],  // Original + CAPABILITIES.md
  result: {
    injectedCapabilities: SelectedCapability[],
    totalTokensUsed: number,
    truncated: boolean,
  }
}
```

### 4. Hook Integration

**Location:** `src/agents/bootstrap-files.ts` (modified)

**Purpose:** Integrate capability injection into bootstrap loading

**Integration point:**
```typescript
export async function resolveBootstrapContextForRun(params) {
  let bootstrapFiles = await resolveBootstrapFilesForRun(params);

  // === NEW: Capability injection ===
  if (config?.capabilities?.enabled !== false) {
    const injectionResult = await injectCapabilities(
      bootstrapFiles,
      {
        recentMessages: await getRecentMessages(params.sessionKey, 5),
        sessionType: determineSessionType(params.sessionKey),
        tokenBudget: config?.capabilities?.maxTokens ?? 1500,
      }
    );
    
    bootstrapFiles = injectionResult.files;
    logInjection(injectionResult);
  }
  // === END capability injection ===

  // Existing hooks continue...
  return buildBootstrapContextFiles(bootstrapFiles, { ... });
}
```

---

## Data Flow

### Bootstrap Loading with Capability Injection

```
resolveBootstrapContextForRun()
    ↓
Load standard bootstrap files (AGENTS.md, SOUL.md, etc.)
    ↓
Get recent messages (last 5)
    ↓
selectRelevantCapabilities()
    ├→ Filter by session type
    ├→ Calculate relevance scores
    ├→ Sort by priority
    └→ Fit within token budget
    ↓
injectCapabilities()
    ├→ Create CAPABILITIES.md (synthetic)
    ├→ Insert after AGENTS.md
    └→ Log injection
    ↓
Existing before_context_build hook (token economy)
    ↓
buildBootstrapContextFiles()
    ↓
Return final context
```

---

## Security Architecture

### Threat Model

**Threats:**
- Malicious capability cards injecting instructions
- Path traversal to load untrusted files
- Prompt injection via capability content

**Mitigations:**

#### 1. File Access Control
```typescript
// Only load from repo root
const repoRoot = path.resolve(__dirname, "../../..");
const capabilityPath = path.join(repoRoot, relativePath);

// Validate path doesn't escape
if (!capabilityPath.startsWith(repoRoot)) {
  throw new Error("Path traversal detected");
}
```

#### 2. Content Sanitization
```typescript
function sanitizeCapabilityContent(content: string): string {
  // Strip frontmatter
  content = stripFrontmatter(content);
  
  // Remove role markers
  content = content.replace(/^(User|Assistant|System):/gim, "[$1]:");
  
  // Remove XML-like tags
  content = content.replace(/<\/?(?:user|assistant|system)>/gi, "");
  
  // Validate no suspicious patterns
  const suspiciousPatterns = [
    /ignore previous instructions/i,
    /disregard.*above/i,
    /new instructions:/i,
  ];
  
  for (const pattern of suspiciousPatterns) {
    if (pattern.test(content)) {
      throw new Error("Suspicious content detected");
    }
  }
  
  return content;
}
```

#### 3. Frontmatter Validation
```typescript
function validateFrontmatter(metadata: any): void {
  // Token budget limits
  if (metadata.token_budget > 2000) {
    throw new Error("Token budget exceeds maximum");
  }
  
  // Type validation
  if (!Array.isArray(metadata.triggers?.keywords)) {
    throw new Error("Invalid triggers.keywords");
  }
  
  // Priority range
  if (metadata.priority < 0 || metadata.priority > 100) {
    throw new Error("Priority out of range");
  }
}
```

### Audit Trail

All injections are logged:

```
[capability-injection] Injected 2 capabilities (850 tokens):
  token-economy (relevance: 0.85, tokens: 600)
  telegram-optimizations (relevance: 0.62, tokens: 250)
```

---

## Performance Considerations

### Time Complexity

**Per request:**
- Keyword scanning: O(M × K) where M = messages, K = keywords
- Relevance scoring: O(C) where C = capabilities
- Sorting: O(C log C)
- File loading: O(S) where S = selected capabilities
- **Total: O(M × K + C log C + S)** - Typically <50ms

### Space Complexity

**Memory usage:**
- Registry: O(C) - ~10-50 capabilities = ~100KB
- Recent messages: O(M) - 5 messages = ~5KB
- Selected capabilities: O(S) - 2-5 capabilities = ~5KB
- **Total: ~110KB** - Negligible

### Token Budget

**Baseline (always loaded):**
- AGENTS.md: ~500 tokens
- Other bootstrap files: ~5000-15000 tokens
- **Capability injection: +400-1500 tokens**

**Token efficiency:**
- Without injection: Agent doesn't know capabilities → multiple clarification turns → +2000-5000 tokens
- With injection: Agent knows capabilities → direct action → net savings

---

## Configuration Schema

```typescript
interface CapabilitiesConfig {
  enabled?: boolean;              // Default: true
  maxTokens?: number;            // Default: 1500
  confidenceThreshold?: number;  // Default: use capability's threshold
  forceInclude?: string[];       // Always inject these IDs
  exclude?: string[];            // Never inject these IDs
  logLevel?: "none" | "summary" | "detailed"; // Default: "summary"
}
```

**Example:**
```json
{
  "capabilities": {
    "enabled": true,
    "maxTokens": 1500,
    "forceInclude": ["token-economy"],
    "exclude": ["experimental-feature"],
    "logLevel": "detailed"
  }
}
```

---

## Extension Points

### Custom Relevance Scoring

Override `calculateRelevanceScore` for custom algorithms:

```typescript
export function calculateRelevanceScore(
  capability: CapabilityMetadata,
  context: SelectionContext,
  customScorer?: (cap: CapabilityMetadata, ctx: SelectionContext) => number,
): number {
  if (customScorer) {
    return customScorer(capability, context);
  }
  
  // Default implementation...
}
```

### Additional Triggers

Extend `CapabilityMetadata.triggers`:

```typescript
triggers: {
  keywords: string[];
  sessionTypes: string[];
  confidenceThreshold: number;
  // NEW: Additional trigger types
  timeOfDay?: string[];        // e.g., ["morning", "evening"]
  userTier?: string[];         // e.g., ["premium", "enterprise"]
  environmentType?: string[];  // e.g., ["development", "production"]
}
```

### Capability Sources

Support external capability sources:

```typescript
interface CapabilitySource {
  type: "local" | "remote" | "database";
  location: string;
  loadCapabilities(): Promise<CapabilityMetadata[]>;
}
```

---

## Testing Strategy

### Unit Tests

**Test coverage:**
- Keyword matching accuracy
- Relevance scoring algorithm
- Token budget enforcement
- Session type filtering
- Content sanitization
- Frontmatter validation

### Integration Tests

**Test scenarios:**
- Injection with various token budgets
- Multiple capabilities selected
- Config overrides (forceInclude, exclude)
- Error handling (missing files, invalid frontmatter)

### End-to-End Tests

**Test flow:**
- Full bootstrap context loading
- Capability injection based on conversation
- Agent uses injected capabilities
- Verify token usage within budget

---

## Failure Modes

### Failure Mode 1: No Relevant Capabilities

**Scenario:** No capabilities match conversation context

**Behavior:** 
- No injection occurs
- Agent uses only AGENTS.md (Tier 1)
- No performance impact

**Mitigation:** Design AGENTS.md to provide baseline awareness

### Failure Mode 2: Token Budget Exceeded

**Scenario:** Selected capabilities exceed token budget

**Behavior:**
- Inject capabilities in priority order
- Stop when budget exhausted
- Log which capabilities were skipped

**Mitigation:** Tune `estimatedTokens` per capability

### Failure Mode 3: File Load Error

**Scenario:** Capability card file missing or unreadable

**Behavior:**
- Skip that capability
- Continue with others
- Log warning

**Mitigation:** Fail-open design (errors don't break startup)

### Failure Mode 4: Invalid Frontmatter

**Scenario:** Capability card has malformed frontmatter

**Behavior:**
- Reject capability
- Log error
- Continue with others

**Mitigation:** Validation at load time

---

## Deployment Considerations

### Gradual Rollout

**Phase 1: Monitoring mode**
- Enable injection
- Log selections
- Don't inject yet
- Analyze logs for 48 hours

**Phase 2: Soft launch**
- Inject for specific session types
- Monitor token usage
- Collect feedback

**Phase 3: Full deployment**
- Enable for all sessions
- Monitor performance
- Tune thresholds

### Performance Monitoring

**Metrics to track:**
- Injection frequency (% of requests)
- Average tokens injected per request
- Selection accuracy (manual review)
- Error rate
- Time to inject (<50ms target)

---

## Design Decisions

### Why Three Tiers?

**Decision:** Three-tier information architecture instead of flat

**Rationale:**
- Tier 1 (AGENTS.md): Minimal always-on cost, maximum compatibility
- Tier 2 (Capability cards): Balance detail vs tokens, only when relevant
- Tier 3 (SYSTEM_ARCHITECTURE.md): Full knowledge available but not automatic

**Alternative considered:** Single large prompt → Rejected (token waste)

### Why Keyword-Based Selection?

**Decision:** Use keyword matching + density for relevance scoring

**Rationale:**
- Fast (<50ms)
- Accurate enough (0.6 threshold catches real relevance)
- Simple to understand and debug
- Easy to tune per capability

**Alternative considered:** LLM-based relevance scoring → Rejected (adds latency + tokens)

### Why Fail-Open?

**Decision:** Errors in capability system don't break agent

**Rationale:**
- Agent startup is critical path
- Capability awareness is enhancement, not requirement
- Better to run without capabilities than not run at all

**Alternative considered:** Fail-closed → Rejected (too risky)

### Why Repo-Controlled Files?

**Decision:** Capability cards must be in repository, not user workspace

**Rationale:**
- Security: Prevents user-injected instructions
- Reliability: Content is version-controlled
- Maintenance: Single source of truth

**Alternative considered:** User-editable cards → Rejected (security risk)

---

## Future Enhancements

### Planned Features

1. **Config-driven base profiles**
   - Minimal/full capability profiles
   - Per-agent configuration

2. **Capability discovery tool**
   - Complementary to injection
   - For runtime capability queries

3. **Dynamic token budgets**
   - Adjust based on conversation length
   - Higher budget for longer contexts

4. **Machine learning relevance**
   - Train model on selection accuracy
   - Improve keyword-based scoring

### Potential Extensions

1. **Multi-language support**
   - Support non-English capability cards
   - Language-specific keyword matching

2. **Capability versioning**
   - Track capability versions
   - Migration paths for breaking changes

3. **A/B testing framework**
   - Test different selection algorithms
   - Measure impact on agent performance

---

## References

- [Implementation Plan](../IMPLEMENTATION_PLAN.md) - Complete implementation guide
- [Summary](../SUMMARY.md) - Quick reference
- [README](../README.md) - Project overview

---

**Last updated:** 2026-02-13  
**Status:** Architecture finalized, ready for implementation
