---
capability_id: unique-identifier-here
version: 1.0
triggers:
  keywords: [keyword1, keyword2, keyword3]
  session_types: [main, isolated]  # Which session types need this
  confidence_threshold: 0.6         # Min relevance score (0.0-1.0)
priority: 10                        # Higher = injected first (0-100)
token_budget: 600                   # Estimated tokens for this card
---

# Capability Name

**Brief one-line description of what this capability does**

## Section 1: Core Features

Describe the main features or functionality.

- Feature 1: Description
- Feature 2: Description
- Feature 3: Description

## Section 2: Usage

Show how to use this capability.

**Example:**
```bash
# Command or code example
some-command --option value
```

## Section 3: Configuration

Where and how to configure this capability.

**Config location:** `path/to/config.json`

**Key settings:**
```json
{
  "setting1": "value1",
  "setting2": "value2"
}
```

## Section 4: Impact/Results

What difference does this capability make?

**Impact:** Brief description of benefits, performance improvement, cost savings, etc.

**Metrics:** Concrete numbers if available

**More info:** Link to detailed documentation (if available)

---

**Notes:**
- Keep total length 200-800 characters (this template is ~1200 chars - trim for production)
- Use clear, concise language
- Focus on actionable information
- Avoid fluff or repetition
- Prioritize information the agent needs to USE the capability
