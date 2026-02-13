# Capability Awareness: Implementation Decision Guide

**Date:** 2026-02-13  
**Status:** Decision Required  
**Options:** Full Injection (6-8h) vs Skills-First (2-3h)

---

## Executive Summary

**Two viable approaches:**

1. **Full Injection** (Technical, 6-8h) - Complete capability awareness system
2. **Skills-First** (Pragmatic, 2-3h) - Leverage existing skills infrastructure

**Recommendation:** Start with Skills-First, upgrade to Full Injection later if needed.

**Rationale:**
- Lower implementation risk
- Faster to deploy (2-3h vs 6-8h)
- Uses proven skills system
- Can upgrade later without breaking changes
- Your history: "build changed → things broke" suggests caution

---

## Option 1: Full Injection Implementation

### What It Is

Complete router-gated capability injection system with:
- Real temp file (`~/.openclaw_runtime/CAPABILITIES.md`)
- Frontmatter-based registry (single source of truth)
- Router-tag selection (zero baseline cost)
- Security hardening (repo-only, path validation)
- Metrics + kill switch

### Implementation Effort

**Total:** 6-8 hours

**Phase 0:** Confirm pipeline (30 min)
- Test if embedder respects real temp files
- Verify workspaceDir is writable
- Confirm `.openclaw_runtime/` can be created

**Phase 1:** Loader (60-90 min)
- Scan `docs/capabilities/*.md`
- Parse frontmatter (gray-matter)
- Validate schema + cache

**Phase 2:** Selector (60-90 min)
- Router tag matching (primary)
- Keyword fallback (secondary)
- Relevance scoring + prioritization

**Phase 3:** Injector (60-90 min)
- Build content
- Enforce token caps
- Write real temp file

**Phase 4:** Integration (60-120 min)
- Modify `bootstrap-files.ts`
- Fail-open error handling
- E2E testing

**Phase 5:** Env surface (60-120 min)
- Create `env.catalog.yml`
- Capability card for env awareness
- Status tool (presence checks only)

**Phase 6:** Metrics (45-90 min)
- Usage tracking
- Kill switch
- Analytics

### Pros

✅ **Automatic** - Agent gets capabilities without explicit action  
✅ **Targeted** - Only when router tags match (zero baseline cost)  
✅ **Secure** - Repo-controlled, validated paths  
✅ **Maintainable** - Content-only updates  
✅ **Monitored** - Metrics + kill switch

### Cons

⚠️ **Complex** - Multiple new modules, integration points  
⚠️ **Risk** - Bootstrap pipeline modification (critical path)  
⚠️ **Testing** - Requires thorough E2E validation  
⚠️ **Debugging** - More moving parts if things break  
⚠️ **Time** - 6-8 hours implementation + testing

### When To Choose

**Choose Full Injection if:**
- You need automatic capability awareness (no agent action required)
- You're comfortable with bootstrap pipeline changes
- You have 6-8 hours for implementation + testing
- You want comprehensive metrics and monitoring
- Token optimization is critical (need zero baseline cost)

---

## Option 2: Skills-First Approach

### What It Is

Leverage existing OpenClaw skills system for capability awareness:

1. **Convert capability cards → skill modules**
   ```
   skills/
   ├── token-economy/
   │   └── SKILL.md
   ├── telegram-primary/
   │   └── SKILL.md
   ├── memory-system/
   │   └── SKILL.md
   └── env-surface/
       └── SKILL.md
   ```

2. **Enhance AGENTS.md with skill hints**
   ```markdown
   # AGENTS.md
   
   This fork has custom capabilities. Before replying, check skills:
   - token-economy: Model routing, context caps, heartbeat optimization
   - telegram-primary: Voice, buttons, TTS integration
   - memory-system: qmd RAG search, embeddings
   - env-surface: Environment presence (not values)
   
   **When relevant:** Read the skill BEFORE acting.
   ```

3. **Optional: Router skill hints**
   ```typescript
   // In router classification
   if (tags.includes('token-economy')) {
     context.recommendedSkill = 'token-economy';
   }
   ```

### Implementation Effort

**Total:** 2-3 hours

**Phase 1:** Create skill modules (60 min)
- Copy capability card content → `skills/*/SKILL.md`
- Add frontmatter (name, description, tags)
- 4-5 skills at ~15 min each

**Phase 2:** Update AGENTS.md (30 min)
- Add skill catalog
- Add "read before acting" instruction
- Keep ≤600 chars

**Phase 3:** Optional router hints (30 min)
- Add `recommendedSkill` to context
- Update system prompt to surface it
- "Recommended: Read token-economy skill"

**Phase 4:** Testing (30-60 min)
- Verify skills appear in `<available_skills>`
- Test agent reads relevant skill
- Confirm descriptions trigger correctly

### Pros

✅ **Simple** - Reuses existing skills system  
✅ **Fast** - 2-3 hours to deploy  
✅ **Safe** - No bootstrap pipeline changes  
✅ **Proven** - Skills system already works  
✅ **Reversible** - Just remove skills, no rollback needed  
✅ **Debuggable** - Easy to see if agent read skill

### Cons

⚠️ **Manual** - Agent must remember to read skill (less reliable)  
⚠️ **Latency** - Extra read tool call (~500ms)  
⚠️ **Token cost** - Small baseline cost (skill descriptions in prompt)  
⚠️ **Less automatic** - Depends on agent following instructions

### When To Choose

**Choose Skills-First if:**
- You want quick deployment (2-3h vs 6-8h)
- You prefer lower implementation risk
- Your history suggests caution with build changes
- You can tolerate small baseline token cost (skill descriptions)
- Agent reliability is acceptable (follows "read skill" instruction)
- You want to defer injection complexity

---

## Detailed Comparison

| Aspect | Full Injection | Skills-First |
|--------|----------------|--------------|
| **Implementation time** | 6-8 hours | 2-3 hours |
| **Code complexity** | High (new modules) | Low (reuse skills) |
| **Bootstrap changes** | Yes (critical path) | No |
| **Baseline token cost** | 0 tokens | ~50-100 tokens (skill descriptions) |
| **Dynamic cost** | +200-600 when triggered | +400-800 when read |
| **Reliability** | High (automatic) | Medium (agent must read) |
| **Debugging** | Complex (multiple modules) | Simple (just skills) |
| **Rollback** | Config/code/git | Just remove skills |
| **Monitoring** | Metrics + kill switch | Basic (skill reads in logs) |
| **Extensibility** | High (designed for it) | Medium (skill system) |

---

## Recommended Path: Hybrid Approach

**Phase 1 (Now):** Skills-First (2-3h)
- Deploy capability awareness via skills
- Measure agent reliability
- Gather data on which capabilities are used

**Phase 2 (Later, if needed):** Upgrade to Injection (6-8h)
- Implement full injection system
- Migrate skills → capability cards
- Keep skills as fallback

**Migration is non-breaking:**
- Skills continue to work
- Injection adds automatic awareness
- Agent can use either path

---

## Decision Criteria

### Choose Skills-First Now If:

1. ✅ Your primary goal is "agent knows capabilities" (done)
2. ✅ You want quick deployment (today vs next week)
3. ✅ You're risk-averse after past build issues
4. ✅ Baseline token cost of ~50-100 is acceptable
5. ✅ You can tolerate agent occasionally forgetting to read skill

### Defer Full Injection If:

1. ⚠️ You don't have 6-8 hours for implementation + testing
2. ⚠️ Bootstrap pipeline changes make you nervous
3. ⚠️ Skills-first meets your immediate needs
4. ⚠️ You want to validate capability content before automating

### Implement Full Injection Now If:

1. ✅ Zero baseline cost is non-negotiable
2. ✅ You need automatic awareness (can't rely on agent reading)
3. ✅ You have time for thorough testing
4. ✅ You want comprehensive metrics/monitoring
5. ✅ You're comfortable with bootstrap changes

---

## Revised Full Injection Plan (If Chosen)

### Critical Corrections Applied

✅ **Real temp file** (no synthetic paths)
```typescript
const runtimeDir = path.join(workspaceDir, '.openclaw_runtime');
await fs.mkdir(runtimeDir, { recursive: true });
const capFilePath = path.join(runtimeDir, 'CAPABILITIES.md');
await fs.writeFile(capFilePath, content, 'utf-8');
```

✅ **Router-gated only** (zero baseline)
```typescript
const routerTags = params.routerTags || [];
const needsCapabilities = routerTags.some(tag => 
  ['config', 'integrations', 'infra', 'deployment', 'memory', 'token_economy', 'debug'].includes(tag)
);

if (!needsCapabilities) {
  return { files: bootstrapFiles, result: { ... } };  // Skip injection
}
```

✅ **Frontmatter-only registry** (no duplication)
```typescript
async function buildRegistry(capabilitiesDir: string): Promise<CapabilityMetadata[]> {
  const files = await fs.readdir(capabilitiesDir);
  return Promise.all(
    files
      .filter(f => f.endsWith('.md') && f !== 'README.md')
      .map(async file => {
        const content = await fs.readFile(path.join(capabilitiesDir, file), 'utf-8');
        const { data } = matter(content);
        return parseCapabilityMetadata(data, file);
      })
  );
}
```

✅ **Safe rollback** (proper JSON editing)
```javascript
// scripts/disable-capabilities.js
const fs = require('fs');
const path = require('path');
const configPath = path.join(process.env.HOME, '.openclaw/openclaw.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
config.capabilities = config.capabilities || {};
config.capabilities.enabled = false;
fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
```

✅ **Repo-only loading** (security-first)
```typescript
async function validateCapabilityPath(filePath: string, repoRoot: string): Promise<void> {
  const capabilitiesDir = path.join(repoRoot, 'docs/capabilities');
  const realPath = await fs.realpath(filePath);
  
  if (!realPath.startsWith(capabilitiesDir + path.sep)) {
    throw new Error('Path outside allowed directory');
  }
  
  const stat = await fs.lstat(filePath);
  if (stat.isSymbolicLink()) {
    throw new Error('Symlinks not allowed');
  }
}
```

### Phase 0: Confirm Pipeline Behavior (30 min)

**Test 1: Embedder accepts temp files**
```typescript
// Test if bootstrap embedder reads from disk
const testFile = path.join(workspaceDir, '.openclaw_runtime/test.md');
await fs.mkdir(path.dirname(testFile), { recursive: true });
await fs.writeFile(testFile, '# Test\n\nContent', 'utf-8');

const testBootstrapFile = {
  name: 'TEST.md' as any,
  path: testFile,
  content: '# Test\n\nContent',
  missing: false,
};

// Add to bootstrap and verify it appears in final context
```

**Test 2: Verify workspaceDir is writable**
```typescript
const runtimeDir = path.join(workspaceDir, '.openclaw_runtime');
await fs.mkdir(runtimeDir, { recursive: true });
const testPath = path.join(runtimeDir, 'write-test.txt');
await fs.writeFile(testPath, 'test', 'utf-8');
await fs.unlink(testPath);
```

**Deliverable:** Confirmation that real temp files work OR identification of blockers

---

## Skills-First Implementation (Recommended Start)

### Phase 1: Create Skill Modules (60 min)

**Structure:**
```
skills/
├── token-economy/
│   └── SKILL.md
├── telegram-primary/
│   └── SKILL.md
├── memory-system/
│   └── SKILL.md
├── second-brain/
│   └── SKILL.md
└── env-surface/
    └── SKILL.md
```

**Template:** `skills/token-economy/SKILL.md`
```markdown
# Token Economy Hooks

**When to use:** Questions about tokens, costs, model routing, context caps

**What exists:**

## Model Routing
- `before_model_select` routes based on task complexity
- GPT-4o (cheap) → Sonnet (balanced) → Opus (complex)
- Check routing: `openclaw status` shows current model

## Context Bundling
- `before_context_build` caps injected context
- Hard limit: 10k tokens
- Filters bootstrap files to fit

## Zero-Token Heartbeat
- Pre-LLM check: if `HEARTBEAT.md` empty → skip API call
- Saves ~50% baseline token spend
- To use: Keep `HEARTBEAT.md` empty when no tasks

## Verification
```bash
# View routing decisions
tail -f ~/.openclaw/token-audit/*.jsonl

# Check current model
openclaw status
```

**Do:**
- ✅ Keep context caps strict
- ✅ Use zero-token heartbeat when possible
- ✅ Check logs to verify routing

**Don't:**
- ❌ Override routing without reason
- ❌ Inject capability cards for casual chat
- ❌ Print token audit logs (may contain sensitive context)

**Deep dive:** See SYSTEM_ARCHITECTURE.md § Token Economy
```

**Create similar skills for:**
- `telegram-primary` - Voice, TTS, buttons, message formatting
- `memory-system` - qmd search, embedding, domain structure
- `second-brain` - Outlook + Todoist integration
- `env-surface` - Environment presence (not values)

### Phase 2: Update AGENTS.md (30 min)

```markdown
# AGENTS.md - OpenClaw Fork Capabilities

This fork has token-economy hooks and production integrations.

**Custom capabilities (check skills before using):**
- token-economy: Model routing, context caps, zero-token heartbeat
- telegram-primary: Voice TTS, inline buttons, formatting
- memory-system: qmd RAG search (57 files, local models)
- second-brain: Outlook + Todoist capture workflows
- env-surface: Check env presence (not values)

**When relevant:** Read the skill BEFORE acting.

**Deep technical details:** SYSTEM_ARCHITECTURE.md (read via tool when needed)

**Config:** ~/.openclaw/openclaw.json  
**Active hooks:** before_model_select, before_context_build
```

### Phase 3: Test (30 min)

**Verify:**
```bash
# Check skills appear
openclaw skills list | grep -E "(token-economy|telegram-primary|memory-system)"

# Test agent reads skill
# Send message: "How does model routing work?"
# Expected: Agent reads token-economy skill, explains routing

# Send message: "How do I use Telegram voice?"
# Expected: Agent reads telegram-primary skill, explains TTS
```

**Metrics:**
- Track skill reads in logs
- Monitor which skills are read most often
- Identify which capabilities need better descriptions

### Phase 4: Iterate (ongoing)

**Tune skill descriptions:**
- If agent doesn't read skill → improve description/keywords
- If agent reads wrong skill → refine descriptions
- If agent reads multiple skills → consolidate related content

**Add new skills as needed:**
- New capability added → create skill in ~15 min
- Update AGENTS.md catalog
- No code changes required

---

## Environment Surface Implementation

**Create:** `docs/env.catalog.yml`
```yaml
# Environment variable catalog (metadata only, no values)
# Purpose: Document what exists, not what it contains

credentials:
  - name: GITHUB_PAT
    purpose: GitHub API access (repo operations)
    required: true
    check: "test -n \"$GITHUB_PAT\""
  
  - name: GROQ_API_KEY
    purpose: Whisper transcription (voice input)
    required: false
    check: "test -n \"$GROQ_API_KEY\""
  
  - name: MICROSOFT_CLIENT_ID
    purpose: Outlook calendar + email (Second Brain)
    required: false
    check: "test -n \"$MICROSOFT_CLIENT_ID\""

ssh_keys:
  - name: GITHUB_PRIVATE_SSH_KEY
    purpose: Git operations via SSH
    location: ~/.ssh/github_ed25519
    check: "test -f ~/.ssh/github_ed25519"

api_tokens:
  - name: TODOIST_API_TOKEN
    purpose: Task management integration
    stored: .env
    check: "grep -q TODOIST_API_TOKEN .env 2>/dev/null"

# Guidelines:
# - Document existence, not values
# - Provide check commands (safe, no echo)
# - Update when adding new integrations
```

**Create:** `skills/env-surface/SKILL.md`
```markdown
# Environment Surface

**When to use:** Questions about configured integrations, API access, credentials

**What to know:**

## Principle: Presence vs Value
- ✅ Check if credential EXISTS
- ❌ Never print or log credential VALUES
- ✅ Use status tools to verify presence
- ❌ Never echo $ENV_VAR or cat files with secrets

## Checking Presence

**Safe commands:**
```bash
# Check if variable is set (no output)
test -n "$GITHUB_PAT" && echo "Present" || echo "Missing"

# Check if file exists
test -f ~/.ssh/github_ed25519 && echo "Present" || echo "Missing"
```

**Status tool:**
```bash
# Use dedicated tool (if available)
openclaw env status
```

## Catalog

**See:** `docs/env.catalog.yml` for complete list

**Common integrations:**
- GitHub: API token + SSH key
- Groq: Whisper transcription
- Microsoft Graph: Outlook calendar/email
- Todoist: Task management
- Telegram: Bot token (via OpenClaw config)

## Do / Don't

**Do:**
- ✅ Check presence when diagnosing issues
- ✅ Use status tools
- ✅ Document new env vars in catalog

**Don't:**
- ❌ Print values (echo $VAR)
- ❌ Log values
- ❌ Include values in diagnostics
- ❌ Share values in group chats

**If credential missing:** Suggest user set it (how-to in catalog), but never ask for value in chat.
```

---

## Migration Path (Skills → Injection)

**When to migrate:**
- Skills-first working well for 1-2 weeks
- Agent reliability measured (>90% reads relevant skill)
- You have 6-8 hours for injection implementation
- Zero baseline cost becomes priority

**Migration steps:**
1. Implement full injection system (phases 0-6)
2. Keep skills as fallback
3. Test injection in parallel
4. Monitor metrics (injection vs skill reads)
5. Gradually rely on injection
6. Keep skills for debugging

**Benefits of hybrid:**
- Injection handles automatic awareness
- Skills provide fallback if injection fails
- Discovery tool + skills = excellent debugging

---

## Final Recommendation

**Start with Skills-First today (2-3h):**
1. Create 5 skill modules
2. Update AGENTS.md
3. Test agent reads skills
4. Deploy to production
5. Monitor for 1-2 weeks

**Decide on Full Injection later:**
- If skills work well → maybe don't need injection
- If agent forgets to read skills → implement injection
- If zero baseline cost critical → implement injection

**Timeline:**
- Today: Skills-first deployment (2-3h)
- Week 1-2: Monitor agent behavior
- Week 3: Decide on injection implementation
- Week 4-5: Implement injection (if chosen)

---

## Questions to Answer Before Proceeding

1. **Phase 0 critical:** Can bootstrap embedder read real temp files?
   - Test with `.openclaw_runtime/test.md`
   - If no → must implement virtual file support (adds complexity)

2. **Router tags:** Are they available at context build time?
   - Check `bootstrap-files.ts` params
   - If no → must pass them or use keyword fallback only

3. **Risk tolerance:** How much complexity are you willing to accept?
   - High → Full injection now
   - Medium → Skills-first now, injection later
   - Low → Skills-first only

4. **Timeline:** When do you need this deployed?
   - Today → Skills-first
   - Next week → Full injection
   - No rush → Skills-first, evaluate, decide

---

**Decision Recommendation:**

Given your context:
- History of "build broke" incidents
- Token economy already sensitive
- Time pressure (project work, meetings)
- Need reliability over optimization

**Choose:** Skills-First approach (today, 2-3h)

**Rationale:**
- Lower risk (no bootstrap changes)
- Faster deployment (usable today)
- Proven system (skills already work)
- Can upgrade later without breaking changes
- Meets core requirement: "agent knows capabilities"

**Defer:** Full injection until skills-first proven for 1-2 weeks

---

**Status:** Awaiting decision

**Next steps:**
1. Choose approach (skills-first recommended)
2. If skills-first: Create 5 skill modules (60 min)
3. If full injection: Run Phase 0 tests (30 min)
4. Deploy chosen approach
5. Monitor and iterate
