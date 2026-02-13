# Capability-Awareness System: Refined Implementation Plan

**Version:** 2.0 (Incorporates Technical Assessment Corrections)  
**Status:** Ready for implementation  
**Estimated effort:** 6-8 hours  
**Token impact:** 0 baseline, +200-600 dynamic (router-gated)

---

## Changes From Original Plan

**Critical corrections applied:**

✅ Real temp file instead of synthetic path  
✅ Frontmatter-only registry (no duplication)  
✅ Router-tag based selection (no getRecentMessages)  
✅ Content-length enforced token caps  
✅ Zero baseline injection (router-gated only)  
✅ Repo-only file loading (security)  
✅ Per-card sanitization (non-blocking)  
✅ Safe JSON editing for rollback  
✅ Metrics instrumentation  
✅ Discovery tool kept as complementary

---

## Part 1: Architecture Overview

### 1.1 Three-Tier System (Unchanged)

**Tier 1: AGENTS.md** (≤600 chars, always loaded)
- Points to capabilities (doesn't describe them)
- References discovery tool
- Minimal token cost

**Tier 2: Capability Cards** (200-800 chars each, router-gated injection)
- Located: `docs/capabilities/*.md`
- Frontmatter: metadata + triggers
- Injected only when router tags match

**Tier 3: Deep Docs** (read via tool, never auto-loaded)
- `SYSTEM_ARCHITECTURE.md`
- Referenced by capability cards
- Agent reads when needed

### 1.2 Injection Flow

```
User message
    ↓
Router classifies (adds tags: cost, integration, etc.)
    ↓
Check if tags match capability triggers
    ↓
  NO → Skip injection (0 tokens)
  YES → Select relevant capabilities
    ↓
Parse frontmatter (build registry in-memory)
    ↓
Calculate relevance scores
    ↓
Sort by priority
    ↓
Write to .openclaw_runtime/CAPABILITIES.md (real file)
    ↓
Inject into bootstrap context
    ↓
Agent gets targeted awareness
```

### 1.3 Token Economy Guarantee

**Zero baseline cost:**
- No injection unless router tags request capabilities
- Casual chat: 0 extra tokens
- Tool use: 0 extra tokens
- Telegram messages: 0 extra tokens

**Dynamic cost:**
- Triggered only by router tags: `config`, `integration`, `deployment`, `memory`, `token-economy`, `debug`
- Cost: +200-600 tokens when triggered
- Frequency: ~5-10% of sessions (estimated)

---

## Part 2: File Structure

### 2.1 Repository Layout

```
openclaw/                           # Fork root
├── AGENTS.md                       # NEW: Ultra-concise manifest (≤600 chars)
├── SYSTEM_ARCHITECTURE.md          # EXISTING: Deep reference
├── docs/
│   ├── capabilities/               # NEW: Capability cards
│   │   ├── README.md               # Index and usage guide
│   │   ├── token-economy.md
│   │   ├── telegram-optimizations.md
│   │   ├── memory-system.md
│   │   └── second-brain-loop.md
│   └── reference/
│       └── ... (existing)
├── src/
│   ├── plugins/
│   │   ├── capabilities/           # NEW: Capability system
│   │   │   ├── index.ts            # Public API
│   │   │   ├── types.ts            # Type definitions
│   │   │   ├── registry.ts         # Frontmatter-based registry
│   │   │   ├── selector.ts         # Tag-based selection
│   │   │   ├── injector.ts         # Safe injection with real files
│   │   │   ├── sanitizer.ts        # Per-card validation
│   │   │   └── metrics.ts          # Logging and analytics
│   │   └── hooks.ts                # EXISTING
│   ├── agents/
│   │   └── bootstrap-files.ts      # MODIFY: Add injection hook
│   └── tools/
│       └── list-capabilities.ts    # NEW: Discovery tool
├── tests/
│   └── plugins/
│       └── capabilities/
│           ├── registry.test.ts
│           ├── selector.test.ts
│           ├── injector.test.ts
│           └── e2e.test.ts
└── scripts/
    └── disable-capabilities.js     # Safe config editing
```

### 2.2 Runtime Files

```
~/.openclaw/
├── openclaw.json                   # Config (with capabilities block)
├── .openclaw_runtime/              # NEW: Runtime temp files
│   └── CAPABILITIES.md             # Injected capability content (real file)
└── capability-metrics.jsonl        # NEW: Injection metrics
```

---

## Part 3: Implementation Details

### 3.1 Types (src/plugins/capabilities/types.ts)

```typescript
export interface CapabilityMetadata {
  id: string;
  name: string;
  description: string;
  filePath: string;
  triggers: {
    routerTags: string[];           // Router classification tags
    keywords: string[];             // Fallback: scan current message
    sessionTypes: string[];         // main, isolated, etc.
    confidenceThreshold: number;
  };
  priority: number;
  maxChars: number;                 // Hard char cap (enforced)
  estimatedTokens?: number;         // For logging only
}

export interface SelectionContext {
  routerTags: string[];             // From router classification
  currentMessage?: string;          // Fallback for keyword matching
  sessionType: string;
  agentId?: string;
  workspaceDir: string;
  repoRoot: string;
  tokenBudget: number;
  config?: CapabilitiesConfig;
}

export interface SelectedCapability {
  metadata: CapabilityMetadata;
  relevanceScore: number;
  content: string;
  tokensUsed: number;               // Actual tokens (approx)
}

export interface CapabilitiesConfig {
  enabled?: boolean;                // Default: true
  maxTokens?: number;               // Default: 1500
  allowWorkspaceCards?: boolean;    // Default: false (security)
  forceInclude?: string[];          // Always include these IDs
  exclude?: string[];               // Never include these IDs
  autoDisableThreshold?: number;    // Kill switch (default: 0.30)
  autoDisableWindowHours?: number;  // Kill switch window (default: 24)
}
```

### 3.2 Registry (src/plugins/capabilities/registry.ts)

**Single source of truth: Markdown frontmatter**

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';
import matter from 'gray-matter';
import { CapabilityMetadata } from './types.js';

let CAPABILITY_REGISTRY_CACHE: CapabilityMetadata[] | null = null;

/**
 * Build capability registry from markdown frontmatter.
 * Cached in memory after first call.
 */
export async function getCapabilityRegistry(repoRoot: string): Promise<CapabilityMetadata[]> {
  if (CAPABILITY_REGISTRY_CACHE) {
    return CAPABILITY_REGISTRY_CACHE;
  }

  const capabilitiesDir = path.join(repoRoot, 'docs/capabilities');
  const files = await fs.readdir(capabilitiesDir);
  const registry: CapabilityMetadata[] = [];

  for (const file of files) {
    if (!file.endsWith('.md') || file === 'README.md') {
      continue;
    }

    try {
      const filePath = path.join(capabilitiesDir, file);
      await validateCapabilityPath(filePath, repoRoot);

      const content = await fs.readFile(filePath, 'utf-8');
      const { data } = matter(content);

      // Validate frontmatter
      validateCapabilityFrontmatter(data, file);

      registry.push({
        id: data.capability_id,
        name: data.name || data.capability_id,
        description: data.description || '',
        filePath: `docs/capabilities/${file}`,
        triggers: {
          routerTags: data.triggers?.routerTags || [],
          keywords: data.triggers?.keywords || [],
          sessionTypes: data.triggers?.sessionTypes || ['main', 'isolated'],
          confidenceThreshold: data.triggers?.confidenceThreshold || 0.6,
        },
        priority: data.priority || 5,
        maxChars: data.max_chars || data.token_budget * 4 || 3200,  // Fallback: estimate
        estimatedTokens: data.token_budget,
      });
    } catch (err) {
      console.warn(`[capability-registry] Skipping ${file}: ${String(err)}`);
      continue;
    }
  }

  CAPABILITY_REGISTRY_CACHE = registry;
  return registry;
}

async function validateCapabilityPath(filePath: string, repoRoot: string): Promise<void> {
  const capabilitiesDir = path.join(repoRoot, 'docs/capabilities');
  const realPath = await fs.realpath(filePath);

  if (!realPath.startsWith(capabilitiesDir)) {
    throw new Error(`Path outside allowed directory: ${realPath}`);
  }

  const stat = await fs.lstat(filePath);
  if (stat.isSymbolicLink()) {
    throw new Error('Symlinks not allowed');
  }

  if (stat.size > 50 * 1024) {
    throw new Error(`File too large: ${stat.size} bytes`);
  }
}

function validateCapabilityFrontmatter(data: any, filename: string): void {
  if (!data.capability_id) {
    throw new Error(`Missing capability_id in ${filename}`);
  }

  if (!data.triggers || !Array.isArray(data.triggers.keywords)) {
    throw new Error(`Invalid triggers.keywords in ${filename}`);
  }

  if (data.token_budget && data.token_budget > 2000) {
    throw new Error(`Token budget exceeds max (2000) in ${filename}`);
  }
}

/**
 * Clear registry cache (for testing)
 */
export function clearRegistryCache(): void {
  CAPABILITY_REGISTRY_CACHE = null;
}
```

### 3.3 Selector (src/plugins/capabilities/selector.ts)

**Router-tag based selection (primary), keyword fallback (secondary)**

```typescript
import { getCapabilityRegistry } from './registry.js';
import { CapabilityMetadata, SelectionContext, SelectedCapability } from './types.js';
import { validateAndSanitizeContent } from './sanitizer.js';
import * as fs from 'fs/promises';
import * as path from 'path';
import matter from 'gray-matter';

export async function selectRelevantCapabilities(
  context: SelectionContext,
  warn?: (msg: string) => void
): Promise<SelectedCapability[]> {
  const registry = await getCapabilityRegistry(context.repoRoot);

  // Filter by config (exclude/forceInclude)
  let candidates = applyConfigFilters(registry, context.config);

  // Filter by session type
  candidates = candidates.filter((cap) =>
    cap.triggers.sessionTypes.includes(context.sessionType)
  );

  // Score by relevance
  const scored = candidates.map((cap) => ({
    capability: cap,
    score: calculateRelevanceScore(cap, context),
  }));

  // Filter by confidence threshold
  const relevant = scored.filter((s) => s.score >= s.capability.triggers.confidenceThreshold);

  if (relevant.length === 0) {
    return [];
  }

  // Sort by priority (high first)
  relevant.sort((a, b) => b.capability.priority - a.capability.priority);

  // Load and validate content, fit within token budget
  const selected: SelectedCapability[] = [];
  let tokensUsed = 0;

  for (const { capability, score } of relevant) {
    if (tokensUsed >= context.tokenBudget) {
      warn?.(`[capability] Token budget exhausted (${tokensUsed}/${context.tokenBudget})`);
      break;
    }

    try {
      const content = await loadCapabilityContent(capability, context.repoRoot);
      const validated = validateAndSanitizeContent(capability.id, content);

      if (!validated.valid) {
        warn?.(`[capability] Skipping ${capability.id}: ${validated.reason}`);
        continue;
      }

      // Enforce max chars
      const truncated = enforceMaxChars(validated.content!, capability.maxChars);
      const actualTokens = Math.ceil(truncated.length / 4);  // Rough estimate

      if (tokensUsed + actualTokens > context.tokenBudget) {
        warn?.(`[capability] Skipping ${capability.id}: would exceed budget`);
        continue;
      }

      selected.push({
        metadata: capability,
        relevanceScore: score,
        content: truncated,
        tokensUsed: actualTokens,
      });

      tokensUsed += actualTokens;
    } catch (err) {
      warn?.(`[capability] Failed to load ${capability.id}: ${String(err)}`);
      continue;
    }
  }

  return selected;
}

function applyConfigFilters(
  registry: CapabilityMetadata[],
  config?: CapabilitiesConfig
): CapabilityMetadata[] {
  let filtered = registry;

  if (config?.exclude?.length) {
    filtered = filtered.filter((cap) => !config.exclude!.includes(cap.id));
  }

  if (config?.forceInclude?.length) {
    // Ensure force-included capabilities are present
    const forceIds = new Set(config.forceInclude);
    const existing = new Set(filtered.map((c) => c.id));
    
    for (const id of forceIds) {
      if (!existing.has(id)) {
        const cap = registry.find((c) => c.id === id);
        if (cap) {
          filtered.push(cap);
        }
      }
    }
  }

  return filtered;
}

function calculateRelevanceScore(
  capability: CapabilityMetadata,
  context: SelectionContext
): number {
  // Primary: Router tags
  if (context.routerTags.length > 0) {
    const matchedTags = capability.triggers.routerTags.filter((tag) =>
      context.routerTags.includes(tag)
    );

    if (matchedTags.length > 0) {
      // Strong signal: router explicitly tagged this
      return 1.0;
    }
  }

  // Fallback: Keyword matching in current message
  if (context.currentMessage) {
    const messageText = context.currentMessage.toLowerCase();
    const matchedKeywords = capability.triggers.keywords.filter((keyword) =>
      messageText.includes(keyword.toLowerCase())
    );

    if (matchedKeywords.length === 0) {
      return 0;
    }

    const keywordRatio = matchedKeywords.length / capability.triggers.keywords.length;
    const matchCount = matchedKeywords.reduce(
      (sum, keyword) => sum + (messageText.match(new RegExp(keyword, 'gi')) || []).length,
      0
    );
    const density = Math.min(matchCount / 50, 1.0);

    return keywordRatio * 0.7 + density * 0.3;
  }

  // No context: use base priority as proxy
  return capability.priority / 20;  // Max priority 10 → 0.5 score
}

async function loadCapabilityContent(
  capability: CapabilityMetadata,
  repoRoot: string
): Promise<string> {
  const filePath = path.join(repoRoot, capability.filePath);
  const content = await fs.readFile(filePath, 'utf-8');
  
  // Strip frontmatter
  const { content: bodyContent } = matter(content);
  return bodyContent.trim();
}

function enforceMaxChars(content: string, maxChars: number): string {
  if (content.length <= maxChars) {
    return content;
  }

  return content.slice(0, maxChars) + '\n\n[Content truncated to fit token budget]';
}
```

### 3.4 Sanitizer (src/plugins/capabilities/sanitizer.ts)

**Per-card validation, non-blocking**

```typescript
export interface ValidationResult {
  valid: boolean;
  content?: string;
  reason?: string;
}

export function validateAndSanitizeContent(
  capabilityId: string,
  content: string
): ValidationResult {
  // Check for suspicious patterns
  const suspiciousPatterns = [
    /ignore (previous|prior|all|above) (instructions|prompts|context)/i,
    /disregard (everything|all|previous|prior) (above|instructions)/i,
    /new instructions:/i,
    /system:\s*you are now/i,
    /forget (everything|all) (you know|about)/i,
  ];

  for (const pattern of suspiciousPatterns) {
    if (pattern.test(content)) {
      return {
        valid: false,
        reason: `Suspicious pattern: ${pattern.source}`,
      };
    }
  }

  // Sanitize content
  let sanitized = content;

  // Remove XML-like role tags
  sanitized = sanitized.replace(/<\/?(?:user|assistant|system)>/gi, '');

  // Escape user role markers
  sanitized = sanitized.replace(/^(User|Assistant|System):/gim, '[$1]:');

  return { valid: true, content: sanitized };
}
```

### 3.5 Injector (src/plugins/capabilities/injector.ts)

**Real temp file, safe injection**

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';
import { selectRelevantCapabilities } from './selector.js';
import { SelectionContext, SelectedCapability } from './types.js';
import { WorkspaceBootstrapFile } from '../agents/workspace.js';

export interface InjectionResult {
  injectedCapabilities: SelectedCapability[];
  totalTokensUsed: number;
  capabilityFilePath?: string;
}

export async function injectCapabilities(
  bootstrapFiles: WorkspaceBootstrapFile[],
  context: SelectionContext,
  warn?: (msg: string) => void
): Promise<{
  files: WorkspaceBootstrapFile[];
  result: InjectionResult;
}> {
  const selected = await selectRelevantCapabilities(context, warn);

  if (selected.length === 0) {
    return {
      files: bootstrapFiles,
      result: { injectedCapabilities: [], totalTokensUsed: 0 },
    };
  }

  // Create runtime directory
  const runtimeDir = path.join(context.workspaceDir, '.openclaw_runtime');
  await fs.mkdir(runtimeDir, { recursive: true });

  // Write capability content to real file
  const capabilityContent = [
    '# Capability Context (auto-injected)',
    '',
    `> Injected ${selected.length} capabilities: ${selected.map((c) => c.metadata.id).join(', ')}`,
    '',
    ...selected.map((cap) => cap.content),
  ].join('\n');

  const capFilePath = path.join(runtimeDir, 'CAPABILITIES.md');
  await fs.writeFile(capFilePath, capabilityContent, 'utf-8');

  // Create bootstrap file entry
  const capabilityFile: WorkspaceBootstrapFile = {
    name: 'CAPABILITIES.md' as any,
    path: capFilePath,  // Real path (not synthetic)
    content: capabilityContent,
    missing: false,
  };

  // Insert after AGENTS.md
  const agentsIndex = bootstrapFiles.findIndex((f) => f.name === 'AGENTS.md');
  const insertIndex = agentsIndex >= 0 ? agentsIndex + 1 : 0;

  const newFiles = [...bootstrapFiles];
  newFiles.splice(insertIndex, 0, capabilityFile);

  const totalTokens = selected.reduce((sum, c) => sum + c.tokensUsed, 0);

  warn?.(
    `[capability-injection] Injected ${selected.length} capabilities (${totalTokens} tokens): ` +
      selected.map((c) => `${c.metadata.id} (${c.relevanceScore.toFixed(2)})`).join(', ')
  );

  return {
    files: newFiles,
    result: {
      injectedCapabilities: selected,
      totalTokensUsed: totalTokens,
      capabilityFilePath: capFilePath,
    },
  };
}
```

### 3.6 Metrics (src/plugins/capabilities/metrics.ts)

**Logging and analytics**

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';

export interface CapabilityMetrics {
  timestamp: number;
  sessionKey: string;
  sessionType: string;
  routerTags: string[];
  injected: Array<{
    id: string;
    relevanceScore: number;
    tokensUsed: number;
  }>;
  totalTokens: number;
  selectionTimeMs: number;
}

export async function logCapabilityMetrics(
  metrics: CapabilityMetrics,
  configDir: string
): Promise<void> {
  try {
    const logPath = path.join(configDir, 'capability-metrics.jsonl');
    const line = JSON.stringify(metrics) + '\n';
    await fs.appendFile(logPath, line, 'utf-8');
  } catch (err) {
    console.warn('[capability-metrics] Failed to log:', err);
  }
}

export async function checkKillSwitch(
  configDir: string,
  threshold: number = 0.30,
  windowHours: number = 24
): Promise<{ shouldDisable: boolean; stats: { triggered: number; total: number } }> {
  try {
    const logPath = path.join(configDir, 'capability-metrics.jsonl');
    const content = await fs.readFile(logPath, 'utf-8');
    const lines = content.trim().split('\n').filter(Boolean);

    const windowMs = windowHours * 60 * 60 * 1000;
    const cutoff = Date.now() - windowMs;

    const recent = lines
      .map((line) => JSON.parse(line) as CapabilityMetrics)
      .filter((m) => m.timestamp >= cutoff);

    const triggered = recent.filter((m) => m.totalTokens > 0).length;
    const total = recent.length;

    const rate = total > 0 ? triggered / total : 0;

    return {
      shouldDisable: rate > threshold && total >= 50,  // Min 50 samples
      stats: { triggered, total },
    };
  } catch (err) {
    return { shouldDisable: false, stats: { triggered: 0, total: 0 } };
  }
}
```

### 3.7 Hook Integration (src/agents/bootstrap-files.ts)

**Modify existing file**

```typescript
// Add imports at top
import { injectCapabilities } from '../plugins/capabilities/injector.js';
import { logCapabilityMetrics, checkKillSwitch } from '../plugins/capabilities/metrics.js';

export async function resolveBootstrapContextForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  agentId?: string;
  routerTags?: string[];        // NEW: From router classification
  currentMessage?: string;      // NEW: For keyword fallback
  warn?: (message: string) => void;
}): Promise<{
  bootstrapFiles: WorkspaceBootstrapFile[];
  contextFiles: EmbeddedContextFile[];
}> {
  let bootstrapFiles = await resolveBootstrapFilesForRun(params);

  // === NEW: Capability injection (BEFORE existing hooks) ===
  const capabilityConfig = params.config?.capabilities;
  if (capabilityConfig?.enabled !== false) {
    try {
      // Check kill switch
      const configDir = path.dirname(params.config?.configPath || '~/.openclaw');
      const killSwitch = await checkKillSwitch(
        configDir,
        capabilityConfig?.autoDisableThreshold,
        capabilityConfig?.autoDisableWindowHours
      );

      if (killSwitch.shouldDisable) {
        params.warn?.(
          `[capability] Auto-disabled: ${killSwitch.stats.triggered}/${killSwitch.stats.total} sessions triggered (>${(capabilityConfig?.autoDisableThreshold || 0.3) * 100}%)`
        );
      } else {
        // Only inject if router tags request capabilities
        const routerTags = params.routerTags || [];
        const needsCapabilities =
          routerTags.some((tag) =>
            ['config', 'integration', 'deployment', 'memory', 'token-economy', 'debug'].includes(
              tag
            )
          ) || capabilityConfig.forceInclude?.length;

        if (needsCapabilities) {
          const startTime = Date.now();
          const repoRoot = path.resolve(__dirname, '../../..');

          const injectionResult = await injectCapabilities(
            bootstrapFiles,
            {
              routerTags,
              currentMessage: params.currentMessage,
              sessionType: isSubagentSessionKey(params.sessionKey) ? 'isolated' : 'main',
              agentId: params.agentId,
              workspaceDir: params.workspaceDir,
              repoRoot,
              tokenBudget: capabilityConfig?.maxTokens ?? 1500,
              config: capabilityConfig,
            },
            params.warn
          );

          bootstrapFiles = injectionResult.files;

          // Log metrics
          await logCapabilityMetrics(
            {
              timestamp: Date.now(),
              sessionKey: params.sessionKey || 'unknown',
              sessionType: isSubagentSessionKey(params.sessionKey) ? 'isolated' : 'main',
              routerTags,
              injected: injectionResult.result.injectedCapabilities.map((c) => ({
                id: c.metadata.id,
                relevanceScore: c.relevanceScore,
                tokensUsed: c.tokensUsed,
              })),
              totalTokens: injectionResult.result.totalTokensUsed,
              selectionTimeMs: Date.now() - startTime,
            },
            configDir
          );
        }
      }
    } catch (err) {
      params.warn?.(`[capability-injection] Failed: ${String(err)}`);
      // Continue with original files (fail-open)
    }
  }
  // === END capability injection ===

  // Existing before_context_build hook continues below...
  // ... rest of function unchanged ...
}
```

---

## Part 4: Capability Cards

### 4.1 Template

```markdown
---
capability_id: example-capability
name: Example Capability
description: Brief description for registry
triggers:
  routerTags: [config, integration]   # Primary: Router tags
  keywords: [example, test, demo]     # Fallback: Keyword scan
  sessionTypes: [main, isolated]
  confidenceThreshold: 0.6
priority: 8
max_chars: 2400                       # Hard cap (enforced)
token_budget: 600                     # Estimate (for logging)
---

# Example Capability

**Brief summary of what this capability does**

## Key Features

- Feature 1
- Feature 2

## Usage

Commands/examples here

## Configuration

Where to configure, what options exist

## References

- Link to SYSTEM_ARCHITECTURE.md section
- Link to external docs
- Command to check status
```

### 4.2 Example: token-economy.md

```markdown
---
capability_id: token-economy
name: Token Economy Hooks
description: Model routing, context bundling, zero-token heartbeat
triggers:
  routerTags: [token-economy, cost, budget, routing]
  keywords: [cost, tokens, model, routing, budget, economy, cheap]
  sessionTypes: [main, isolated]
  confidenceThreshold: 0.6
priority: 10
max_chars: 2400
token_budget: 600
---

# Token Economy Hooks

**Active optimizations in this fork:**

## Model Routing (before_model_select)

**Strategy:** Cheap-first with escalation
- GPT-4o (cheap) → Sonnet (balanced) → Opus (expensive)
- Based on task complexity scoring
- Config: `routing.cheapFirst: true`

**Check current model:**
```bash
# View session status
openclaw status
```

## Context Bundling (before_context_build)

**Strategy:** Hard token caps
- Max 10k tokens per context bundle
- Filters bootstrap files to fit budget
- Prioritizes recent/relevant files

**Audit trail:**
```bash
# View token usage
tail -f ~/.openclaw/token-audit/*.jsonl
```

## Zero-Token Heartbeat

**Strategy:** Pre-LLM check
- If `HEARTBEAT.md` empty → skip API call
- 100% elimination of heartbeat token cost
- Saves ~50% of baseline token spend

**Configure:**
```markdown
# HEARTBEAT.md (workspace)
# Keep empty to skip all heartbeat calls
# Add tasks to run periodic checks
```

## Impact

**Expected savings:** 60-80% token reduction
**Cost:** ~$1-1.50/day (vs $3-5/day without hooks)
**Monthly:** ~$30-45/month savings

**View real-time stats:**
```bash
# Check token economy status
openclaw token-economy status
```

## Configuration

**Location:** `~/.openclaw/openclaw.json`

```json
{
  "routing": {
    "cheapFirst": true,
    "maxContextTokens": 10000
  }
}
```

## References

- See SYSTEM_ARCHITECTURE.md § Token Economy Implementation
- Audit logs: `~/.openclaw/token-audit/`
- Cron job: Token economy monitor (daily at 2 AM)
```

### 4.3 AGENTS.md (Repo Root)

```markdown
# AGENTS.md - OpenClaw Fork Capabilities

This fork has token-economy hooks and production integrations.

**Capability domains:**
- token-economy: Model routing + context bundling + zero-token heartbeat
- telegram: Primary channel (voice, inline buttons, TTS)
- memory-system: RAG-based semantic memory (qmd + local models)
- second-brain: Outlook + Todoist integration

**Discovery:**
- Use `list_capabilities` tool to see what's available
- Router auto-injects relevant capabilities when needed
- Read SYSTEM_ARCHITECTURE.md for deep technical details

**Active hooks:** before_model_select, before_context_build  
**Config:** ~/.openclaw/openclaw.json
```

---

## Part 5: Discovery Tool

### 5.1 Implementation (src/tools/list-capabilities.ts)

```typescript
import { getCapabilityRegistry } from '../plugins/capabilities/registry.js';
import * as path from 'path';

export async function listCapabilitiesTool(params: {
  detail?: 'summary' | 'full';
  workspaceDir: string;
}): Promise<string> {
  const repoRoot = path.resolve(__dirname, '../..');
  const registry = await getCapabilityRegistry(repoRoot);

  if (params.detail === 'full') {
    return JSON.stringify(registry, null, 2);
  }

  // Summary view
  const summary = registry.map((cap) => ({
    id: cap.id,
    name: cap.name,
    description: cap.description,
    tags: cap.triggers.routerTags,
    priority: cap.priority,
  }));

  return (
    '**Available Capabilities:**\n\n' +
    summary
      .sort((a, b) => b.priority - a.priority)
      .map(
        (cap) =>
          `- **${cap.name}** (${cap.id})\n  ${cap.description}\n  Tags: ${cap.tags.join(', ')}`
      )
      .join('\n\n')
  );
}
```

---

## Part 6: Configuration & Rollback

### 6.1 Config Schema

```json
{
  "capabilities": {
    "enabled": true,
    "maxTokens": 1500,
    "allowWorkspaceCards": false,
    "forceInclude": ["token-economy"],
    "exclude": [],
    "autoDisableThreshold": 0.30,
    "autoDisableWindowHours": 24
  }
}
```

### 6.2 Safe Rollback Script (scripts/disable-capabilities.js)

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');

const configPath = path.join(process.env.HOME, '.openclaw/openclaw.json');
const backupPath = configPath + '.bak';

try {
  // Backup
  fs.copyFileSync(configPath, backupPath);
  console.log(`Backed up config to ${backupPath}`);

  // Load
  const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));

  // Modify
  config.capabilities = config.capabilities || {};
  config.capabilities.enabled = false;

  // Write
  fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
  console.log('Disabled capabilities in config');

  console.log('\nNext steps:');
  console.log('  docker compose restart openclaw-gateway');
} catch (err) {
  console.error('Failed to disable capabilities:', err.message);
  if (fs.existsSync(backupPath)) {
    console.log(`Restore backup: cp ${backupPath} ${configPath}`);
  }
  process.exit(1);
}
```

**Usage:**
```bash
node scripts/disable-capabilities.js
docker compose restart openclaw-gateway
```

---

## Part 7: Implementation Phases

### Phase 1: Infrastructure (2 hours)

**Goal:** Core types and registry

**Tasks:**
1. Create `src/plugins/capabilities/types.ts`
2. Create `src/plugins/capabilities/registry.ts`
3. Create `src/plugins/capabilities/sanitizer.ts`
4. Write unit tests for registry

**Validation:**
```bash
npm test -- capabilities/registry.test.ts
```

### Phase 2: Selection & Injection (2.5 hours)

**Goal:** Tag-based selection and safe injection

**Tasks:**
1. Create `src/plugins/capabilities/selector.ts`
2. Create `src/plugins/capabilities/injector.ts`
3. Create `src/plugins/capabilities/metrics.ts`
4. Write unit tests

**Validation:**
```bash
npm test -- capabilities/selector.test.ts
npm test -- capabilities/injector.test.ts
```

### Phase 3: Hook Integration (1.5 hours)

**Goal:** Integrate with bootstrap pipeline

**Tasks:**
1. Modify `src/agents/bootstrap-files.ts`
2. Add config schema
3. Create discovery tool
4. Write integration tests

**Validation:**
```bash
npm run build
npm test -- capabilities/e2e.test.ts
```

### Phase 4: Content & Documentation (2 hours)

**Goal:** Capability cards and docs

**Tasks:**
1. Create `docs/capabilities/` structure
2. Write 4-5 capability cards
3. Create `AGENTS.md`
4. Create rollback script
5. Update SYSTEM_ARCHITECTURE.md

**Validation:**
```bash
# Test registry building
node -e "require('./dist/plugins/capabilities/registry.js').getCapabilityRegistry('/app').then(r => console.log(r.length + ' capabilities'))"
```

---

## Part 8: Testing Strategy

### Unit Tests

**registry.test.ts:**
- Parse frontmatter correctly
- Validate required fields
- Reject invalid paths
- Cache behavior

**selector.test.ts:**
- Router tag matching
- Keyword fallback
- Token budget enforcement
- Priority sorting

**injector.test.ts:**
- Real file creation
- Bootstrap file insertion
- Error handling (fail-open)
- Token cap enforcement

### Integration Test

**e2e.test.ts:**
```typescript
it('should inject capabilities when router tags match', async () => {
  const result = await resolveBootstrapContextForRun({
    workspaceDir: '/tmp/test-workspace',
    routerTags: ['token-economy', 'config'],
    sessionType: 'main',
    config: { capabilities: { enabled: true, maxTokens: 1500 } },
  });

  expect(result.bootstrapFiles.some(f => f.name === 'CAPABILITIES.md')).toBe(true);
  expect(fs.existsSync('/tmp/test-workspace/.openclaw_runtime/CAPABILITIES.md')).toBe(true);
});

it('should NOT inject when router tags do not match', async () => {
  const result = await resolveBootstrapContextForRun({
    workspaceDir: '/tmp/test-workspace',
    routerTags: ['casual'],
    sessionType: 'main',
    config: { capabilities: { enabled: true } },
  });

  expect(result.bootstrapFiles.some(f => f.name === 'CAPABILITIES.md')).toBe(false);
});
```

### Manual Testing

```bash
# Start gateway with capability system
npm run build
npm run gateway:start

# Test case 1: Router tag match
# Send message that triggers "config" or "token-economy" classification
# Verify injection in logs

# Test case 2: No tag match
# Send casual message
# Verify NO injection (0 baseline cost)

# Test case 3: Force-include
# Set config.capabilities.forceInclude: ["telegram-optimizations"]
# Send any message
# Verify injection happens

# Test case 4: Kill switch
# Simulate >30% injection rate over 24h
# Verify auto-disable triggers
```

---

## Part 9: Success Criteria

**Before PR approval:**

- [ ] All unit tests pass
- [ ] Integration test passes
- [ ] E2E test with real gateway passes
- [ ] Manual testing confirms:
  - Zero baseline cost (no injection without router tags)
  - Injection works when router tags match
  - Kill switch triggers correctly
  - Rollback script works
- [ ] Token impact measured:
  - Baseline: 0 tokens (confirmed)
  - Dynamic: +200-600 tokens (when triggered)
- [ ] Documentation complete:
  - REFINED_IMPLEMENTATION_PLAN.md (this file)
  - TECHNICAL_ASSESSMENT.md (corrections doc)
  - ARCHITECTURE.md (design decisions)
  - Capability cards (4-5 initial)
  - AGENTS.md (repo root)
- [ ] Security validated:
  - Repo-only file loading
  - Path validation works
  - Symlink rejection works
  - Size caps enforced
- [ ] Metrics instrumentation:
  - capability-metrics.jsonl logging
  - Kill switch functional

---

## Part 10: Rollback Procedures

### Level 1: Config Disable (Instant)

```bash
node scripts/disable-capabilities.js
docker compose restart openclaw-gateway
```

### Level 2: Code Disable (5 minutes)

```typescript
// src/agents/bootstrap-files.ts
// Comment out capability injection block
/*
const capabilityConfig = params.config?.capabilities;
if (capabilityConfig?.enabled !== false) {
  // ... entire block ...
}
*/
```

```bash
npm run build
docker compose restart openclaw-gateway
```

### Level 3: Git Revert (10 minutes)

```bash
git log --oneline --graph | head -10
git revert <commit-hash>
npm run build
docker compose restart openclaw-gateway
```

### Validation After Rollback

```bash
# Verify no injection
docker logs openclaw-gateway | grep capability
# Should see: no matches or "capabilities.enabled: false"

# Test conversation
# Should work identically to pre-implementation
```

---

## Part 11: Monitoring & Maintenance

### Metrics to Track

**Daily (automated):**
```bash
# Injection rate
cat ~/.openclaw/capability-metrics.jsonl | \
  jq -s 'map(select(.timestamp > (now - 86400))) | {triggered: map(select(.totalTokens > 0)) | length, total: length, rate: (map(select(.totalTokens > 0)) | length) / length}'

# Avg tokens per injection
cat ~/.openclaw/capability-metrics.jsonl | \
  jq -s 'map(select(.totalTokens > 0)) | map(.totalTokens) | add / length'

# Most common capabilities
cat ~/.openclaw/capability-metrics.jsonl | \
  jq -s '[.[].injected[].id] | group_by(.) | map({cap: .[0], count: length}) | sort_by(.count) | reverse | .[0:5]'
```

**Weekly (manual):**
- Review injection patterns
- Check if new capabilities needed
- Validate token impact vs expectations
- Update capability cards if content drifted

### Capability Card Maintenance

**When to update:**
- Feature changes (command syntax, config keys)
- New capabilities added
- Deprecated features removed
- Hard-coded numbers become stale

**How to update:**
1. Edit `docs/capabilities/<id>.md`
2. Update frontmatter if needed
3. Clear registry cache (restart gateway)
4. Test injection

**No code changes needed for content updates.**

---

## Part 12: Expected Outcomes

### Token Impact

**Baseline:** 0 tokens (router-gated, no always-on injection)  
**Dynamic:** +200-600 tokens when triggered  
**Frequency:** ~5-10% of sessions (estimated)  
**Net monthly cost:** +$3-6/month (vs $0 baseline)  
**Net benefit:** Better routing = fewer retries = net neutral or positive

### Reliability Improvement

✅ Agent consistently aware of token-economy hooks  
✅ Understands Telegram-specific features  
✅ Knows when to leverage memory system  
✅ Aware of available security controls  
✅ Better model selection (fewer escalations)

### Maintenance

**New capability:** Add card + test (~30 min)  
**Update capability:** Edit card (~10 min)  
**Debug:** Use `list_capabilities` tool + metrics logs

---

## Next Steps

1. ✅ Review technical assessment (completed)
2. ✅ Review refined implementation plan (this document)
3. ▶️ **Implement Phase 1** (infrastructure)
4. ▶️ Implement Phase 2 (selection & injection)
5. ▶️ Implement Phase 3 (hook integration)
6. ▶️ Implement Phase 4 (content & docs)
7. ▶️ Test thoroughly (unit + integration + E2E)
8. ▶️ Create PR to fork
9. ▶️ Deploy to production
10. ▶️ Monitor for 48 hours
11. ▶️ Iterate based on metrics

---

**End of Refined Implementation Plan**

**Changes from v1.0:**
- ✅ Real temp file instead of synthetic
- ✅ Frontmatter-only registry
- ✅ Router-tag based selection
- ✅ Content-length token caps
- ✅ Zero baseline injection
- ✅ Repo-only file loading
- ✅ Per-card sanitization
- ✅ Safe JSON rollback
- ✅ Metrics instrumentation
- ✅ Discovery tool kept

**Status:** Ready for implementation

**Approval:** Technical assessment corrections applied
