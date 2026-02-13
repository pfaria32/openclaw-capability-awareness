# Capability Awareness System - Current Status

**Last Updated:** 2026-02-13 05:05 UTC  
**Version:** 2.0 (Refined + Decision Guide)  
**Repository:** https://github.com/pfaria32/Capability-Awareness-System.git (pending push)

---

## 📋 Summary

Router-gated just-in-time capability injection system for AI agent forks, with comprehensive technical assessment and two implementation paths.

---

## 📁 Documentation Complete

### Core Documents (v2.0)

✅ **TECHNICAL_ASSESSMENT.md** (15KB)
- Independent architectural review
- 12 critical issues identified
- Security vulnerabilities found
- Token-economy regressions detected
- Unsafe procedures corrected

✅ **REFINED_IMPLEMENTATION_PLAN.md** (36KB)
- Complete corrected implementation
- Real temp files (no synthetic paths)
- Frontmatter-only registry
- Router-gated selection (zero baseline)
- Content-length token caps
- Repo-only loading
- Safe rollback procedures
- Metrics instrumentation

✅ **REFINED_SUMMARY.md** (9KB)
- Executive summary
- Key changes v1.0 → v2.0
- Token economics (0 baseline)
- Security model
- Success criteria

✅ **ARCHITECTURE.md** (19KB)
- Design decisions and rationale
- Security model (threat model + controls)
- Token economics analysis
- Alternative approaches evaluated
- Tradeoffs documented
- Extension points

✅ **IMPLEMENTATION_DECISION.md** (19KB)
- Two implementation paths
- Skills-First (2-3h, recommended)
- Full Injection (6-8h, advanced)
- Decision criteria
- Migration path
- Environment surface guidance

### Supporting Files

✅ **README.md** - Overview + version notice  
✅ **examples/capability-card-example.md** - Annotated example  
✅ **templates/capability-card-template.md** - Quick-start template  
✅ **templates/AGENTS.md-template.md** - Repo manifest  
✅ **templates/config-template.json** - Config options

---

## 🎯 Implementation Paths

### Option 1: Skills-First (Recommended)

**Effort:** 2-3 hours  
**Risk:** Low  
**Baseline cost:** ~50-100 tokens (skill descriptions)

**What:**
- Convert capability cards → skill modules
- Enhance AGENTS.md with skill hints
- Optional router skill recommendations
- Test agent reads relevant skills

**Why:**
- Reuses proven skills system
- No bootstrap pipeline changes
- Fast deployment (today)
- Can upgrade to injection later

**When to choose:**
- Need quick deployment
- Risk-averse after build issues
- Small baseline cost acceptable
- Can tolerate agent occasionally forgetting

### Option 2: Full Injection

**Effort:** 6-8 hours  
**Risk:** Medium  
**Baseline cost:** 0 tokens (router-gated)

**What:**
- Complete capability injection system
- Real temp file writing
- Frontmatter-based registry
- Router-tag selection
- Metrics + kill switch

**Why:**
- Zero baseline cost (critical for token-economy)
- Automatic awareness (no agent action needed)
- Comprehensive monitoring
- Designed for extensibility

**When to choose:**
- Zero baseline cost non-negotiable
- Need automatic awareness
- Have 6-8 hours for thorough testing
- Comfortable with bootstrap changes

---

## ✅ Decision Recommendation

**Start with Skills-First:**

1. Lower implementation risk
2. Faster deployment (2-3h vs 6-8h)
3. Uses proven infrastructure
4. Can upgrade later without breaking
5. Aligns with "build broke" history

**Timeline:**
- Today: Deploy skills-first (2-3h)
- Week 1-2: Monitor agent behavior
- Week 3: Decide on injection upgrade
- Week 4-5: Implement injection (if needed)

---

## 📊 Key Changes (v1.0 → v2.0)

### Architectural

✅ Real temp files (was: synthetic paths)  
✅ Zero baseline cost (was: +400 tokens)  
✅ Router-gated only (was: always inject minimal)  
✅ Frontmatter-only registry (was: TS + markdown duplication)  
✅ Content-length caps (was: metadata estimates only)

### Security

✅ Repo-only loading (was: workspace fallback by default)  
✅ Path validation (was: basic prefix check)  
✅ Per-card sanitization (was: global throw on error)  
✅ Safe rollback (was: cat >> JSON corruption)

### Implementation

✅ Skills-first alternative (was: injection only)  
✅ Phase 0 confirmation (was: assume pipeline works)  
✅ Environment surface (was: not addressed)  
✅ Migration path (was: not documented)

---

## 🔒 Security Model

**Trust Boundary:**
- Repo-controlled files: Trusted
- Workspace files: Untrusted (opt-in only)

**Controls:**
- Realpath validation + prefix check
- Symlink rejection
- Size caps (50KB max per card)
- Content sanitization (per-card, non-blocking)
- Audit logging (JSONL)

**Attack Surface:**
- File read only (validated paths)
- No network, eval, or exec
- Content sanitized (not executed)

---

## 💰 Token Economics

### Skills-First

**Baseline:** ~50-100 tokens (skill descriptions in `<available_skills>`)  
**Dynamic:** +400-800 tokens (when agent reads skill)  
**Frequency:** ~10-20% of sessions  
**Monthly cost:** ~$3-5

### Full Injection

**Baseline:** 0 tokens (router-gated)  
**Dynamic:** +200-600 tokens (when router tags match)  
**Frequency:** ~5-10% of sessions  
**Monthly cost:** ~$2-4

**Net benefit (both):**
- Better capability awareness → fewer retries
- Estimated savings: $10-15/month from reduced confusion

---

## 📦 Git History

**Commit Timeline:**

1. `8dfba9bcb` - Initial v1.0 plan (CAPABILITY_AWARENESS_PLAN.md)
2. `78a518f` - v2.0 refined implementation (technical assessment applied)
3. `4ddbcc6` - Implementation decision guide (skills-first vs injection)

**Branch:** `main`  
**Remote:** Not pushed yet (awaiting decision + push from host)

---

## 🚀 Next Steps

### Immediate (Today)

1. **Decide implementation approach:**
   - Skills-First (recommended) → 2-3h deployment
   - Full Injection → 6-8h implementation

2. **If Skills-First chosen:**
   - Create 5 skill modules (60 min)
   - Update AGENTS.md (30 min)
   - Test agent behavior (30 min)
   - Deploy to production (immediate)

3. **If Full Injection chosen:**
   - Run Phase 0 tests (30 min)
   - Implement Phase 1-6 (6-8h)
   - Test thoroughly (2-3h)
   - Deploy with monitoring

### Short-term (Week 1-2)

- Monitor agent behavior
- Track capability usage
- Identify which skills/capabilities used most
- Tune descriptions/keywords

### Medium-term (Week 3-4)

- Decide on injection upgrade (if skills-first)
- Implement injection (if chosen)
- Migrate skills → capability cards
- Test hybrid approach

### Long-term

- Push to GitHub: https://github.com/pfaria32/Capability-Awareness-System.git
- Share with OpenClaw community
- Document lessons learned
- Consider upstreaming (if applicable)

---

## 📝 Open Questions

1. **Phase 0 Critical:** Can bootstrap embedder read real temp files?
   - Test: `.openclaw_runtime/test.md`
   - If no → must implement virtual file support

2. **Router Tags:** Available at context build time?
   - Check: `bootstrap-files.ts` params
   - If no → use keyword fallback only

3. **Risk Tolerance:** How much complexity acceptable?
   - High → Full injection now
   - Medium → Skills-first, evaluate, upgrade
   - Low → Skills-first only

4. **Timeline:** When deployed?
   - Today → Skills-first only option
   - Next week → Full injection possible
   - No rush → Skills-first, evaluate for 2 weeks

---

## 🎓 Lessons Learned

### Technical Assessment Value

- Independent review caught 12 critical issues
- Synthetic paths would have silently failed
- Baseline injection conflicted with token-economy
- Registry duplication guaranteed drift
- Unsafe rollback would corrupt config

### Implementation Philosophy

- **Start simple:** Skills-first proves concept
- **Upgrade later:** Full injection when needed
- **Test assumptions:** Phase 0 confirms pipeline
- **Fail-open:** Errors don't break agent
- **Security-first:** Repo-only, validated paths

### Project Management

- **Document decisions:** Clear rationale for choices
- **Provide alternatives:** Skills-first vs injection
- **Measure twice:** Technical assessment before implementation
- **Pragmatic over perfect:** Ship skills-first, upgrade later

---

## 📚 References

**Internal:**
- `IMPLEMENTATION_DECISION.md` - Choose approach
- `REFINED_IMPLEMENTATION_PLAN.md` - Full injection guide
- `TECHNICAL_ASSESSMENT.md` - What changed and why
- `ARCHITECTURE.md` - Design decisions

**External:**
- OpenClaw docs: /app/docs
- Skills system: /app/docs/skills.md
- Bootstrap files: /app/src/agents/bootstrap-files.ts
- Router classification: /app/src/routing/

---

## ✅ Ready State

**Documentation:** Complete (5 core docs + examples + templates)  
**Technical Review:** Complete (12 issues corrected)  
**Implementation Paths:** Defined (skills-first + full injection)  
**Decision Guidance:** Clear (skills-first recommended)  
**Git Status:** Committed (3 commits), not pushed  
**Next Action:** Decide approach, implement chosen path

---

**Status:** ✅ Ready for implementation

**Awaiting:** Decision on Skills-First vs Full Injection

**Recommendation:** Skills-First (2-3h, lower risk, proven system)
