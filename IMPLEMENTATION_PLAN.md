# Capability-Awareness System: Implementation Plan

**Goal:** Make the agent reliably aware of custom fork capabilities without bloating the always-on system prompt, optimized for token economy.

**Status:** PR-ready implementation plan  
**Estimated effort:** 4-6 hours  
**Token impact:** Net negative (saves tokens via smarter injection)

---

## Executive Summary

### Current State

- ✅ Bootstrap files (AGENTS.md, SOUL.md, etc.) are loaded for every session
- ✅ Skills live in `skills/*/SKILL.md` and are lazy-loaded when relevant
- ✅ `before_context_build` hook exists for filtering bootstrap files
- ❌ No structured capability registry
- ❌ No smart injection based on conversation context
- ❌ Fork-specific capabilities (token-economy hooks, Telegram optimizations) are not documented in-prompt

### Proposed Solution

**Three-tier capability awareness system:**

1. **AGENTS.md (Repo root)** - Ultra-concise (≤500 chars) "bootstrap manifest"
   - Points to detailed docs (don't include them)
   - Lists available capability domains
   - Loaded always, minimal token cost

2. **docs/capabilities/** - Structured capability cards (200-800 chars each)
   - One file per capability domain
   - YAML frontmatter for metadata + selection logic
   - Injected just-in-time via `before_context_build` hook

3. **SYSTEM_ARCHITECTURE.md** - Deep technical reference (existing)
   - Never loaded automatically
   - Agent reads via `read` tool when needed

**Token budget:** Max 1500 tokens for capability injection (configurable)

---

## Part 1: Current System Survey

### 1.1 How Bootstrap Files Are Consumed

**Discovery:** `src/agents/workspace.ts`

```typescript
export const DEFAULT_AGENTS_FILENAME = "AGENTS.md";
export const DEFAULT_SOUL_FILENAME = "SOUL.md";
export const DEFAULT_TOOLS_FILENAME = "TOOLS.md";
// ... etc
```

**Loading:** `src/agents/bootstrap-files.ts`

```typescript
export async function resolveBootstrapContextForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  // ...
}): Promise<{
  bootstrapFiles: WorkspaceBootstrapFile[];
  contextFiles: EmbeddedContextFile[];
}>;
```

**Hook point:** Already has `before_context_build` hook!

```typescript
// === TOKEN ECONOMY: before_context_build hook ===
const hookRunner = getGlobalHookRunner();
if (hookRunner?.hasHooks("before_context_build")) {
  const hookResult = await hookRunner.runBeforeContextBuild(
    { requestedFiles: bootstrapFiles.map(...) },
    { agentId, sessionKey, workspaceDir, config }
  );

  if (hookResult?.filteredFiles) {
    // Filter bootstrap files based on hook result
    bootstrapFiles = bootstrapFiles.filter((f) => filteredPaths.has(f.path));
  }
}
// === END TOKEN ECONOMY ===
```

**Max chars:** `resolveBootstrapMaxChars(config)` → defaults to 20,000 chars

### 1.2 How Skills Are Consumed

**Discovery:** `src/agents/skills.ts` (inferred)

- Skills found in `<workspaceDir>/skills/*/SKILL.md`
- Frontmatter parsed for metadata (name, description, tags, etc.)

**Presentation:** `src/agents/system-prompt.ts`

```typescript
return [
  "## Skills (mandatory)",
  "Before replying: scan <available_skills> <description> entries.",
  `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${params.readToolName}\`, then follow it.`,
  "- If multiple could apply: choose the most specific one, then read/follow it.",
  "- If none clearly apply: do not read any SKILL.md.",
  "Constraints: never read more than one skill up front; only read after selecting.",
  trimmed, // <available_skills> list
];
```

**Current pattern:** Lazy loading - agent reads SKILL.md when relevant

### 1.3 Existing Hook Infrastructure

**Hook types:** `src/plugins/types.ts`

```typescript
export interface PluginHooks {
  before_model_select?: BeforeModelSelectHook;
  before_context_build?: BeforeContextBuildHook;
  // ... potentially more
}
```

**BeforeContextBuildHook:**

```typescript
export interface BeforeContextBuildHook {
  (
    params: {
      requestedFiles: Array<{ path: string; type: "bootstrap" | "skill" | "other" }>;
    },
    context: HookContext,
  ): Promise<BeforeContextBuildResult | void>;
}

export interface BeforeContextBuildResult {
  filteredFiles?: Array<{ path: string; type: string }>;
  reason?: string;
}
```

**Current usage:** Token economy uses this to cap context size

---

## Part 2: Proposed File/Folder Layout

### 2.1 Repository Structure

```
openclaw/                           # Your fork root
├── AGENTS.md                       # NEW: Ultra-concise bootstrap manifest (≤500 chars)
├── SYSTEM_ARCHITECTURE.md          # EXISTING: Deep technical docs (not auto-loaded)
├── docs/
│   ├── capabilities/               # NEW: Structured capability cards
│   │   ├── README.md               # Index and usage guide
│   │   ├── token-economy.md        # Model routing, context bundling, etc.
│   │   ├── telegram-optimizations.md
│   │   ├── memory-system.md
│   │   ├── second-brain-loop.md
│   │   ├── security-controls.md
│   │   └── custom-hooks.md         # Hook registry and usage
│   └── reference/
│       └── ... (existing docs)
├── src/
│   ├── plugins/
│   │   ├── capabilities/           # NEW: Capability injection logic
│   │   │   ├── index.ts            # Main capability loader
│   │   │   ├── selector.ts         # Context-based selection logic
│   │   │   ├── registry.ts         # Capability metadata registry
│   │   │   └── injector.ts         # Token-aware injection
│   │   └── hooks.ts                # EXISTING: Hook runner
│   └── agents/
│       └── bootstrap-files.ts      # MODIFY: Integrate capability injection
└── tests/
    └── plugins/
        └── capabilities/            # NEW: Test suite
            ├── selector.test.ts
            └── injector.test.ts
```

### 2.2 AGENTS.md Template (Repo Root)

**Purpose:** Ultra-concise "what's special about this instance"  
**Token budget:** ≤500 chars  
**Always loaded:** Yes

```markdown
# AGENTS.md - OpenClaw Fork Capabilities

This fork has token-economy hooks and Telegram optimizations.

**Capability domains:**

- token-economy: Model routing, context bundling, zero-token heartbeat
- telegram: Primary channel, inline buttons, voice support
- memory-system: RAG-based semantic memory (qmd + local models)
- second-brain: Outlook + Todoist integration
- security: OpenClaw Shield (optional tier-0 controls)

**Deep docs:** See SYSTEM_ARCHITECTURE.md (read via tool when needed)  
**Capability cards:** Auto-injected when relevant (docs/capabilities/)

**Active hooks:** before_model_select, before_context_build  
**Config:** ~/.openclaw/openclaw.json
```

**Key principles:**

- ≤500 chars (enforced)
- Points, doesn't explain
- Always loaded (low token cost)
- Updated manually when adding capabilities

### 2.3 Capability Card Template

**Location:** `docs/capabilities/*.md`  
**Token budget:** 200-800 chars each  
**Loaded:** Just-in-time based on context

**Example:** `docs/capabilities/token-economy.md`

```markdown
---
capability_id: token-economy
version: 1.0
triggers:
  keywords: [cost, tokens, model, routing, budget, economy]
  session_types: [main, isolated]
  confidence_threshold: 0.6
priority: 10
token_budget: 600
---

# Token Economy Hooks

**Active optimizations:**

## Model Routing (before_model_select)

- GPT-4o → Sonnet → Opus based on task complexity
- Config: `routing.cheapFirst: true`
- Routing logic in hooks

## Context Bundling (before_context_build)

- Hard cap: 10k tokens per context bundle
- Filters bootstrap files to fit budget
- Logged to token audit trail

## Zero-Token Heartbeat

- Pre-LLM check: if HEARTBEAT.md empty → skip API call
- Saves ~50% of token spend
- 100% heartbeat cost elimination

**Impact:** 60-80% token reduction, ~$60-105/month savings

**Config location:** `~/.openclaw/openclaw.json`  
**Audit logs:** `~/.openclaw/token-audit/*.jsonl`
```

**Frontmatter fields:**

- `capability_id`: Unique identifier
- `version`: Semantic version
- `triggers.keywords`: Words that trigger injection
- `triggers.session_types`: Which sessions need this
- `triggers.confidence_threshold`: Relevance score cutoff
- `priority`: Higher = injected first (if token budget allows)
- `token_budget`: Estimated tokens for this card

**Selection logic:**

- Scan recent conversation for trigger keywords
- Check session type match
- Calculate relevance score
- Sort by priority
- Inject until token budget exhausted

### 2.4 Capability Registry

**Location:** `src/plugins/capabilities/registry.ts`

**Purpose:** Central metadata for all capabilities

```typescript
export interface CapabilityMetadata {
  id: string;
  name: string;
  description: string;
  filePath: string;
  triggers: {
    keywords: string[];
    sessionTypes: string[];
    confidenceThreshold: number;
  };
  priority: number;
  estimatedTokens: number;
}

export const CAPABILITY_REGISTRY: CapabilityMetadata[] = [
  {
    id: "token-economy",
    name: "Token Economy Hooks",
    description: "Model routing, context bundling, zero-token heartbeat",
    filePath: "docs/capabilities/token-economy.md",
    triggers: {
      keywords: ["cost", "tokens", "model", "routing", "budget", "economy"],
      sessionTypes: ["main", "isolated"],
      confidenceThreshold: 0.6,
    },
    priority: 10,
    estimatedTokens: 600,
  },
  // ... more capabilities
];
```

---

## Part 3: Just-In-Time Injection Implementation

### 3.1 Capability Selector

**Location:** `src/plugins/capabilities/selector.ts`

**Purpose:** Decide which capability cards to inject based on context

```typescript
export interface SelectionContext {
  recentMessages?: string[]; // Last N messages (for keyword matching)
  sessionType: string; // main, isolated, etc.
  agentId?: string;
  workspaceDir: string;
  tokenBudget: number; // Max tokens for capability injection
}

export interface SelectedCapability {
  metadata: CapabilityMetadata;
  relevanceScore: number;
  content: string;
}

export async function selectRelevantCapabilities(
  context: SelectionContext,
): Promise<SelectedCapability[]> {
  const candidates = CAPABILITY_REGISTRY.filter((cap) =>
    cap.triggers.sessionTypes.includes(context.sessionType),
  );

  const scored = candidates.map((cap) => ({
    capability: cap,
    score: calculateRelevanceScore(cap, context),
  }));

  // Filter by confidence threshold
  const relevant = scored.filter((s) => s.score >= s.capability.triggers.confidenceThreshold);

  // Sort by priority (high first)
  relevant.sort((a, b) => b.capability.priority - a.capability.priority);

  // Load content and fit within token budget
  const selected: SelectedCapability[] = [];
  let tokensUsed = 0;

  for (const { capability, score } of relevant) {
    if (tokensUsed + capability.estimatedTokens > context.tokenBudget) {
      break; // Budget exhausted
    }

    const content = await loadCapabilityContent(capability.filePath, context.workspaceDir);
    if (!content) {
      continue; // File missing or unreadable
    }

    selected.push({
      metadata: capability,
      relevanceScore: score,
      content,
    });

    tokensUsed += capability.estimatedTokens;
  }

  return selected;
}

function calculateRelevanceScore(
  capability: CapabilityMetadata,
  context: SelectionContext,
): number {
  if (!context.recentMessages || context.recentMessages.length === 0) {
    // No conversation context - use base priority as proxy
    return capability.priority / 100;
  }

  const recentText = context.recentMessages.join(" ").toLowerCase();
  const matchedKeywords = capability.triggers.keywords.filter((keyword) =>
    recentText.includes(keyword.toLowerCase()),
  );

  if (matchedKeywords.length === 0) {
    return 0;
  }

  // Score = (matched keywords / total keywords) * keyword density
  const keywordRatio = matchedKeywords.length / capability.triggers.keywords.length;
  const matchCount = matchedKeywords.reduce(
    (sum, keyword) => sum + (recentText.match(new RegExp(keyword, "gi")) || []).length,
    0,
  );
  const density = Math.min(matchCount / 100, 1.0); // Cap at 1.0

  return keywordRatio * 0.7 + density * 0.3;
}

async function loadCapabilityContent(
  relativePath: string,
  workspaceDir: string,
): Promise<string | null> {
  try {
    // Resolve path: first try workspace, then repo root
    const repoRoot = path.resolve(__dirname, "../../.."); // Adjust as needed
    const candidates = [path.join(workspaceDir, relativePath), path.join(repoRoot, relativePath)];

    for (const candidate of candidates) {
      try {
        const content = await fs.readFile(candidate, "utf-8");
        return stripFrontmatter(content); // Remove YAML frontmatter
      } catch {
        continue;
      }
    }

    return null;
  } catch (err) {
    console.warn(`Failed to load capability: ${relativePath}`, err);
    return null;
  }
}
```

### 3.2 Capability Injector

**Location:** `src/plugins/capabilities/injector.ts`

**Purpose:** Inject selected capabilities into bootstrap context

```typescript
export interface InjectionResult {
  injectedCapabilities: SelectedCapability[];
  totalTokensUsed: number;
  truncated: boolean;
}

export async function injectCapabilities(
  bootstrapFiles: WorkspaceBootstrapFile[],
  context: SelectionContext,
): Promise<{
  files: WorkspaceBootstrapFile[];
  result: InjectionResult;
}> {
  const selected = await selectRelevantCapabilities(context);

  if (selected.length === 0) {
    return {
      files: bootstrapFiles,
      result: { injectedCapabilities: [], totalTokensUsed: 0, truncated: false },
    };
  }

  // Create synthetic bootstrap file for capabilities
  const capabilityContent = [
    "# Capability Context (auto-injected)",
    "",
    "The following capabilities are active in this fork:",
    "",
    ...selected.map((cap) => cap.content),
  ].join("\n");

  const capabilityFile: WorkspaceBootstrapFile = {
    name: "CAPABILITIES.md" as any, // Synthetic file
    path: "<synthetic>/CAPABILITIES.md",
    content: capabilityContent,
    missing: false,
  };

  // Insert after AGENTS.md but before other files
  const agentsIndex = bootstrapFiles.findIndex((f) => f.name === "AGENTS.md");
  const insertIndex = agentsIndex >= 0 ? agentsIndex + 1 : 0;

  const newFiles = [...bootstrapFiles];
  newFiles.splice(insertIndex, 0, capabilityFile);

  return {
    files: newFiles,
    result: {
      injectedCapabilities: selected,
      totalTokensUsed: selected.reduce((sum, c) => sum + c.metadata.estimatedTokens, 0),
      truncated: false, // Could implement truncation if needed
    },
  };
}
```

### 3.3 Hook Integration

**Location:** `src/agents/bootstrap-files.ts` (modify existing)

**Changes:**

```typescript
// Add import at top
import { injectCapabilities } from "../plugins/capabilities/injector.js";
import { getRecentMessages } from "../routing/session-history.js"; // Hypothetical

export async function resolveBootstrapContextForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  sessionId?: string;
  agentId?: string;
  warn?: (message: string) => void;
}): Promise<{
  bootstrapFiles: WorkspaceBootstrapFile[];
  contextFiles: EmbeddedContextFile[];
}> {
  let bootstrapFiles = await resolveBootstrapFilesForRun(params);

  // === NEW: Capability injection (before existing hooks) ===
  const capabilityConfig = params.config?.capabilities;
  if (capabilityConfig?.enabled !== false) {
    try {
      const recentMessages = await getRecentMessages(params.sessionKey, 5); // Last 5 messages
      const tokenBudget = capabilityConfig?.maxTokens ?? 1500;

      const injectionResult = await injectCapabilities(bootstrapFiles, {
        recentMessages,
        sessionType: isSubagentSessionKey(params.sessionKey) ? "isolated" : "main",
        agentId: params.agentId,
        workspaceDir: params.workspaceDir,
        tokenBudget,
      });

      bootstrapFiles = injectionResult.files;

      if (injectionResult.result.injectedCapabilities.length > 0) {
        params.warn?.(
          `[capability-injection] Injected ${injectionResult.result.injectedCapabilities.length} capabilities (${injectionResult.result.totalTokensUsed} tokens): ${injectionResult.result.injectedCapabilities.map((c) => c.metadata.id).join(", ")}`,
        );
      }
    } catch (err) {
      params.warn?.(`[capability-injection] Failed: ${String(err)}`);
      // Continue with original files on error (fail-open)
    }
  }
  // === END capability injection ===

  // Existing before_context_build hook continues below...
  const hookRunner = getGlobalHookRunner();
  if (hookRunner?.hasHooks("before_context_build")) {
    // ... existing code ...
  }

  const contextFiles = buildBootstrapContextFiles(bootstrapFiles, {
    maxChars: resolveBootstrapMaxChars(params.config),
    warn: params.warn,
  });

  return { bootstrapFiles, contextFiles };
}
```

### 3.4 Configuration

**Location:** `src/config/config.ts` (add to OpenClawConfig)

```typescript
export interface OpenClawConfig {
  // ... existing fields
  capabilities?: {
    enabled?: boolean; // Default: true
    maxTokens?: number; // Default: 1500
    confidenceThreshold?: number; // Default: 0.6 (use capability's threshold)
    forceInclude?: string[]; // Always include these capability IDs
    exclude?: string[]; // Never include these capability IDs
  };
}
```

**User config example:** `~/.openclaw/openclaw.json`

```json
{
  "capabilities": {
    "enabled": true,
    "maxTokens": 1500,
    "forceInclude": ["token-economy"],
    "exclude": ["security-controls"]
  }
}
```

### 3.5 Security: Prompt Injection Prevention

**Risk:** Malicious capability cards could inject instructions

**Mitigations:**

1. **Capability files are repo-controlled** (not user-editable)
   - Only load from `<repoRoot>/docs/capabilities/`
   - Fallback to workspace only for testing/dev
   - Validate path doesn't escape directory

2. **Content sanitization:**

```typescript
function sanitizeCapabilityContent(content: string): string {
  // Strip any remaining frontmatter (should be done already)
  content = stripFrontmatter(content);

  // Remove any user role markers (prevent role confusion)
  content = content.replace(/^(User|Assistant|System):/gim, "[$1]:");

  // Remove XML-like tags that could confuse parsing
  content = content.replace(/<\/?(?:user|assistant|system)>/gi, "");

  // Validate no suspicious patterns
  const suspiciousPatterns = [
    /ignore previous instructions/i,
    /disregard.*above/i,
    /new instructions:/i,
  ];

  for (const pattern of suspiciousPatterns) {
    if (pattern.test(content)) {
      throw new Error("Capability content contains suspicious instructions");
    }
  }

  return content;
}
```

3. **Frontmatter validation:**

```typescript
function validateCapabilityFrontmatter(metadata: any): void {
  // Ensure token budget is reasonable
  if (metadata.token_budget > 2000) {
    throw new Error("Capability token budget exceeds maximum (2000)");
  }

  // Validate triggers
  if (!Array.isArray(metadata.triggers?.keywords)) {
    throw new Error("Invalid triggers.keywords");
  }

  // ... more validation
}
```

4. **Logging:**

```typescript
// Log all injections for audit
params.warn?.(
  `[capability-injection] Files: ${injectionResult.result.injectedCapabilities.map((c) => `${c.metadata.id} (${c.relevanceScore.toFixed(2)})`).join(", ")}`,
);
```

---

## Part 4: Implementation Plan

### 4.1 Phase 1: Infrastructure (2 hours)

**Goal:** Set up types, registry, and selection logic

**Tasks:**

1. **Create type definitions**
   - File: `src/plugins/capabilities/types.ts`
   - Export: `CapabilityMetadata`, `SelectionContext`, `SelectedCapability`, etc.

2. **Create registry**
   - File: `src/plugins/capabilities/registry.ts`
   - Start with 3-5 capability definitions (token-economy, telegram, memory)

3. **Implement selector**
   - File: `src/plugins/capabilities/selector.ts`
   - Functions: `selectRelevantCapabilities`, `calculateRelevanceScore`, `loadCapabilityContent`

4. **Write unit tests**
   - File: `tests/plugins/capabilities/selector.test.ts`
   - Test cases:
     - Keyword matching
     - Session type filtering
     - Token budget enforcement
     - Relevance scoring

**Validation:**

```bash
npm test -- capabilities/selector.test.ts
```

### 4.2 Phase 2: Injection Logic (1.5 hours)

**Goal:** Implement injector and hook integration

**Tasks:**

1. **Implement injector**
   - File: `src/plugins/capabilities/injector.ts`
   - Function: `injectCapabilities`
   - Security: Add `sanitizeCapabilityContent`

2. **Integrate with bootstrap-files.ts**
   - Modify: `src/agents/bootstrap-files.ts`
   - Add capability injection before existing hooks
   - Add error handling (fail-open)

3. **Add config support**
   - Modify: `src/config/config.ts`
   - Add `CapabilitiesConfig` interface
   - Validate in config loader

4. **Write integration tests**
   - File: `tests/plugins/capabilities/injector.test.ts`
   - Test cases:
     - Successful injection
     - Token budget overflow
     - File loading errors
     - Config override behavior

**Validation:**

```bash
npm test -- capabilities/injector.test.ts
npm run build
```

### 4.3 Phase 3: Capability Cards (1.5 hours)

**Goal:** Create initial capability cards

**Tasks:**

1. **Create directory structure**

   ```bash
   mkdir -p docs/capabilities
   ```

2. **Write capability cards**
   - `docs/capabilities/token-economy.md` (600 tokens)
   - `docs/capabilities/telegram-optimizations.md` (400 tokens)
   - `docs/capabilities/memory-system.md` (500 tokens)
   - `docs/capabilities/README.md` (index + usage guide)

3. **Create AGENTS.md**
   - File: `AGENTS.md` (repo root)
   - Ultra-concise manifest (≤500 chars)

4. **Update registry**
   - Add all capability cards to `registry.ts`
   - Ensure token budgets are accurate

**Validation:**

```bash
# Manually verify frontmatter parsing
node -e "console.log(require('gray-matter')(require('fs').readFileSync('docs/capabilities/token-economy.md', 'utf-8')))"
```

### 4.4 Phase 4: Testing & Refinement (1 hour)

**Goal:** End-to-end testing and tuning

**Tasks:**

1. **E2E test**
   - File: `tests/agents/bootstrap-files.capability-injection.e2e.test.ts`
   - Test full flow: conversation → selection → injection → context build

2. **Manual testing**

   ```bash
   # Start gateway with capability injection
   npm run gateway:start

   # Test conversations with trigger keywords
   # Verify injected capabilities in logs
   ```

3. **Tune parameters**
   - Adjust `confidenceThreshold` values
   - Verify `estimatedTokens` accuracy
   - Test token budget limits (500, 1000, 1500)

4. **Performance check**
   - Measure injection overhead (<50ms target)
   - Profile `selectRelevantCapabilities` function

**Validation:**

```bash
npm test -- capability-injection.e2e.test.ts
npm run test:integration
```

### 4.5 Code Touchpoints Summary

**New files:**

- `src/plugins/capabilities/index.ts`
- `src/plugins/capabilities/types.ts`
- `src/plugins/capabilities/registry.ts`
- `src/plugins/capabilities/selector.ts`
- `src/plugins/capabilities/injector.ts`
- `tests/plugins/capabilities/selector.test.ts`
- `tests/plugins/capabilities/injector.test.ts`
- `tests/agents/bootstrap-files.capability-injection.e2e.test.ts`
- `docs/capabilities/README.md`
- `docs/capabilities/token-economy.md`
- `docs/capabilities/telegram-optimizations.md`
- `docs/capabilities/memory-system.md`
- `AGENTS.md` (repo root)

**Modified files:**

- `src/agents/bootstrap-files.ts` (add injection hook)
- `src/config/config.ts` (add CapabilitiesConfig)
- `src/config/schema.json` (add config schema)

**Total:** ~15 new files, 3 modified files, ~1200 lines of code

---

## Part 5: Rollback Plan

### 5.1 Configuration-Based Disable

**Quick disable (no code changes):**

```bash
# On host
cat >> ~/.openclaw/openclaw.json << 'EOF'
{
  "capabilities": {
    "enabled": false
  }
}
EOF

# Restart gateway
docker compose restart openclaw-gateway
```

**Effect:** Injection skipped, bootstrap files load normally

### 5.2 Code-Level Disable

**If config disable doesn't work:**

1. **Comment out hook in bootstrap-files.ts:**

   ```typescript
   // === DISABLED: Capability injection ===
   // const capabilityConfig = params.config?.capabilities;
   // if (capabilityConfig?.enabled !== false) {
   //   ... injection code ...
   // }
   // === END DISABLED ===
   ```

2. **Rebuild and redeploy:**
   ```bash
   npm run build
   docker compose restart
   ```

### 5.3 Git Revert

**Complete rollback:**

```bash
# Identify commit before capability system
git log --oneline --graph | head -10

# Revert to previous commit
git revert <commit-hash>

# Or hard reset (if not pushed)
git reset --hard <commit-hash>

# Rebuild
npm run build
docker compose restart
```

### 5.4 Validation After Rollback

```bash
# Verify agent starts normally
docker logs openclaw-gateway | grep -i capability

# Test basic conversation
# Should work identically to before implementation
```

---

## Part 6: Alternative Approaches

### 6.1 Capability Discovery Tool

**Concept:** Instead of injecting content, expose a `list_capabilities` tool that agent can call

**Pros:**

- Zero token cost until agent explicitly requests
- Agent decides what's relevant (more dynamic)
- No injection complexity

**Cons:**

- Agent must remember to call tool (less reliable)
- Adds tool call latency (~1-2 seconds)
- Requires agent to scan list and decide (more thinking tokens)

**Verdict:** ❌ **Not recommended**

- Adds latency to every "capabilities question"
- Requires agent to be proactive (unreliable)
- Our goal is to make agent "just know" capabilities

### 6.2 Runtime Tool Registry

**Concept:** Expose fork-specific tools (e.g., `check_token_economy_stats`) that reveal capabilities through usage

**Pros:**

- Capabilities discovered through action (self-documenting)
- Zero prompt tokens
- Encourages hands-on learning

**Cons:**

- Agent must discover tools organically (slow)
- Doesn't help with "why is this different from vanilla OpenClaw"
- Telegram-specific behaviors not discoverable via tools

**Verdict:** ✅ **Complementary**

- Use alongside capability injection
- Good for operational stats (token usage, routing decisions)
- Recommend: Add `token_economy_status` tool

### 6.3 Config-Driven Injection (Proposed Alternative)

**Concept:** Instead of keyword-based selection, use session/agent config to pre-define capability profiles

**Example config:**

```json
{
  "agents": {
    "main": {
      "capabilityProfile": "full",
      "capabilities": ["token-economy", "telegram", "memory", "second-brain"]
    },
    "subagents": {
      "capabilityProfile": "minimal",
      "capabilities": ["token-economy"]
    }
  }
}
```

**Pros:**

- Predictable (no heuristics)
- Faster (no keyword scanning)
- Explicit control per agent/session

**Cons:**

- Less dynamic (can't adapt to conversation)
- Requires manual configuration
- Higher baseline token cost (always injects profile)

**Verdict:** ✅ **Hybrid approach recommended**

**Recommendation:** Combine both approaches:

1. **Config-driven base profile** (always injected)
   - `capabilityProfile: "minimal"` → Only critical capabilities
   - `capabilityProfile: "full"` → All capabilities

2. **Keyword-based augmentation** (optional, adds relevant extras)
   - If user asks about tokens → add `token-economy` (if not in base)
   - If user asks about Telegram → add `telegram-optimizations`

**Implementation change:**

```typescript
const baseProfile = config?.agents?.[agentId]?.capabilityProfile ?? "minimal";
const baseCapabilities = getProfileCapabilities(baseProfile);

const keywordSelected = await selectRelevantCapabilities(context);

// Merge: base + keyword (dedup by ID)
const merged = deduplicateCapabilities([...baseCapabilities, ...keywordSelected]);
```

**Token impact:**

- Minimal profile: ~400 tokens (just token-economy + core)
- Full profile: ~1500 tokens (all capabilities)
- Keyword augmentation: +200-600 tokens (situational)

---

## Part 7: Recommendation & Next Steps

### 7.1 Recommended Approach

**Primary:** Just-in-time capability injection (as designed in Parts 1-5)

**Enhancements:**

1. Add config-driven base profiles (minimal/full)
2. Add `token_economy_status` tool (complementary)
3. Keep SYSTEM_ARCHITECTURE.md as deep reference (read via tool)

### 7.2 Implementation Priority

**Phase 1 (Core):** 4-6 hours

- Capability selection + injection
- 3-5 initial capability cards
- Unit + integration tests

**Phase 2 (Enhancements):** 2-3 hours

- Config-driven profiles
- `token_economy_status` tool
- More capability cards

**Phase 3 (Polish):** 1-2 hours

- Performance tuning
- Documentation
- Monitor token impact in production

### 7.3 Expected Outcomes

**Token impact:**

- Baseline: +400 tokens (AGENTS.md + minimal profile)
- Dynamic: +200-600 tokens (keyword-triggered)
- **Net neutral or positive** (smarter context = better routing = fewer retries)

**Reliability improvement:**

- Agent consistently aware of token-economy hooks
- Understands Telegram-specific features
- Knows when to check memory system
- Aware of available security controls

**Maintenance:**

- New capabilities: Add card + registry entry (~15 min)
- Update capabilities: Edit card content (~5 min)
- No code changes needed for content updates

### 7.4 PR Checklist

**Before PR:**

- [ ] All unit tests pass
- [ ] Integration tests pass
- [ ] E2E test passes
- [ ] Manual testing in local gateway
- [ ] Token impact measured (<1500 tokens baseline)
- [ ] Documentation complete (README, capability cards)
- [ ] Config schema updated
- [ ] Rollback tested

**PR Structure:**

```
feat: add capability-awareness system

- Implement just-in-time capability injection
- Add capability registry and selection logic
- Create initial capability cards (token-economy, telegram, memory)
- Add config-driven capability profiles
- Add rollback mechanism

Token impact: +400 baseline, +200-600 dynamic (net neutral)
Performance: <50ms injection overhead

Fixes: Agent awareness of fork-specific features
```

---

## Part 8: Quick Start Implementation

**To implement immediately:**

1. **Create capability infrastructure:**

   ```bash
   mkdir -p src/plugins/capabilities
   mkdir -p tests/plugins/capabilities
   mkdir -p docs/capabilities
   ```

2. **Copy templates from this plan:**
   - `registry.ts` (Part 3.1)
   - `selector.ts` (Part 3.1)
   - `injector.ts` (Part 3.2)
   - `types.ts` (Part 3.1)

3. **Modify bootstrap-files.ts:**
   - Add injection hook (Part 3.3)

4. **Create capability cards:**
   - `token-economy.md` (Part 2.3 example)
   - `telegram-optimizations.md`
   - `memory-system.md`

5. **Create AGENTS.md:**
   - Use template from Part 2.2

6. **Test:**

   ```bash
   npm run build
   npm test
   npm run gateway:start
   ```

7. **Tune:**
   - Adjust `estimatedTokens` based on actual usage
   - Refine `confidenceThreshold` per capability
   - Monitor logs for injection patterns

---

## Appendix: Example Capability Cards

### A.1 telegram-optimizations.md

```markdown
---
capability_id: telegram-optimizations
version: 1.0
triggers:
  keywords: [telegram, inline, buttons, voice, audio, keyboard, chat, message, channel]
  session_types: [main]
  confidence_threshold: 0.5
priority: 8
token_budget: 400
---

# Telegram Optimizations

**Primary channel:** Telegram (via openclaw-telegram plugin)

**Available features:**

- Inline buttons: Set `webchat.capabilities.inlineButtons` for button support
- Voice TTS: Enabled, use [[tts:...]] tags for natural speech
- Voice input: Automatic transcription via Whisper (Groq)
- Reply context: Use [[reply_to_current]] tag
- Silent replies: Use NO_REPLY to skip response

**TTS config:**

- Auto-summary if >1500 chars
- Only use TTS when user sends voice/audio
- Keep spoken text concise

**Button patterns:**

- Not enabled by default for webchat
- Ask to set `webchat.capabilities.inlineButtons` if needed
- Format: Standard Telegram inline keyboard JSON

**Platform limitations:**

- No markdown tables (use bullet lists)
- Headers work but keep short
- Links are native (no wrapping needed)
```

### A.2 memory-system.md

````markdown
---
capability_id: memory-system
version: 1.0
triggers:
  keywords: [memory, remember, recall, search, qmd, rag, semantic, vector, knowledge]
  session_types: [main, isolated]
  confidence_threshold: 0.6
priority: 9
token_budget: 500
---

# RAG-Based Memory System

**Implementation:** qmd + local models (embeddinggemma-300M)

**Structure:**

- 57 files indexed, 76 chunks embedded
- Domain-first: work/family/personal/system
- Types: daily, projects, people, decisions, learnings, procedures

**Search modes:**

- BM25: <100ms (keyword, use by default)
- Vector: ~1-5s (semantic)
- Hybrid: ~3-10s (best quality)

**Commands:**

```bash
# Quick search
export PATH="$HOME/.bun/bin:$PATH"
qmd search "query" -n 5 -c memory

# Helper script
bash /home/node/.openclaw/workspace/memory/ops/scripts/query-memory.sh search "query"
```
````

**When to query:**

- User asks "do you remember..."
- Questions about people, projects, dates
- Context needed for past decisions
- "What did we decide about..."

**Skill location:** `/skills/memory-query/SKILL.md`

**Automation:**

- Daily embeddings: 2 AM UTC
- Daily rollups: 11 PM UTC
- Weekly promotions: Sunday 8 AM

**Cost:** $0/month (all local models)

````

### A.3 second-brain-loop.md

```markdown
---
capability_id: second-brain-loop
version: 1.0
triggers:
  keywords: [outlook, email, calendar, todoist, task, event, meeting, inbox]
  session_types: [main]
  confidence_threshold: 0.7
priority: 7
token_budget: 350
---

# Second Brain Loop Integration

**Outlook integration:**
- Account: bobtheclaw@outlook.com
- Check email: `node projects/second-brain-loop/src/cli/check-outlook.js email`
- Check calendar: `node projects/second-brain-loop/src/cli/check-outlook.js calendar`
- Upcoming events: `node projects/second-brain-loop/src/cli/check-outlook.js upcoming 7`

**Todoist integration:**
- Projects: Inbox, Shopping, Project Ideas
- Helper: `source scripts/todoist-helpers.sh`
- Commands: `todoist_tasks`, `todoist_add`, `todoist_complete`

**Capture workflows:**
- Outlook → memory: `bash memory/ops/scripts/capture-from-outlook.sh`
- Todoist → memory: `bash memory/ops/scripts/capture-from-todoist.sh`

**Note:** Tasks are ephemeral. Store outcomes/decisions in memory, not raw tasks.

**Project Ideas sync:** Keep `memory/project-ideas.md` and Todoist project aligned.
````

---

**End of Implementation Plan**

**Next steps:**

1. Review plan with team/Pedro
2. Implement Phase 1 (infrastructure)
3. Create PR with initial capability cards
4. Test in production for 48 hours
5. Iterate based on logs and usage patterns

**Questions?** See Part 6 for alternative approaches or Part 5 for rollback options.
