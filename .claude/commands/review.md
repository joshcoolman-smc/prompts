---
description: Review code for issues and improvements
---

Review this code: $ARGUMENTS

## What I Look For

**Structure**
- Is this doing too much? Should it be split?
- Is logic in the right place (data, business, UI)?
- Are there obvious extractions waiting to happen?

**Quality**
- Type safety (no `any` types, proper interfaces)
- Error handling and edge cases
- Loading states and user feedback
- Dead code or debug artifacts

**Architecture**
- Does it follow the patterns in `docs/APPROACH.md`?
- Repository-Service-Hooks pattern where appropriate?
- Feature organized or scattered?

**Pragmatic Concerns**
- Is this over-engineered for what it does?
- Is it under-engineered for its complexity?
- Will the next person understand this?

## Output Format

**Critical** - Fix now
**Suggested** - Should address
**Optional** - Nice to have
**Good** - What's working well

Provide specific, actionable feedback. Don't nitpick formatting.
