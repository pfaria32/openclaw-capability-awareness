---
capability_id: example-capability
name: Example Capability
description: Brief one-line description for registry
triggers:
  routerTags: [example, demo, test]     # Primary: Router classification tags
  keywords: [example, test, demo, foo]   # Fallback: Current message scan
  sessionTypes: [main, isolated]         # Which session types need this
  confidenceThreshold: 0.6               # Min relevance score (0-1)
priority: 5                              # Higher = injected first (1-10)
max_chars: 2400                          # Hard cap (enforced via content truncation)
token_budget: 600                        # Estimate (for logging/metrics only)
---

# Example Capability

**Brief summary (1-2 sentences) of what this capability provides**

## Key Features

- Feature 1: Description
- Feature 2: Description
- Feature 3: Description

## Usage

**Commands:**
```bash
# Example command 1
example-command --flag value

# Example command 2
example-tool status
```

**API:**
```typescript
// Example code
const result = await exampleFunction();
```

## Configuration

**Location:** `~/.openclaw/openclaw.json`

```json
{
  "example": {
    "enabled": true,
    "option": "value"
  }
}
```

**Environment variables:**
```bash
export EXAMPLE_API_KEY="your-key-here"
```

## Status Checks

**Check if enabled:**
```bash
# View current status
openclaw example status
```

**View logs:**
```bash
# Tail example logs
tail -f ~/.openclaw/example.log
```

## Impact

**Expected benefit:** Describe measurable impact (time saved, cost reduction, etc.)

**Cost:** Describe any costs (API calls, tokens, etc.)

## References

- [Deep Documentation](../docs/SYSTEM_ARCHITECTURE.md#example-section)
- [External Docs](https://example.com/docs)
- [Source Code](https://github.com/example/repo)

---

**Last Updated:** 2026-02-13  
**Maintainer:** Your Name

**Notes:**
- Keep content concise (aim for 200-800 chars total)
- Reference live data sources instead of hard-coding numbers
- Use present tense ("This capability provides..." not "will provide")
- Focus on what the agent needs to know to USE the capability
