---
description: Start a new feature with clarifying questions
---

I need help implementing: $ARGUMENTS

## Before We Start

Ask me about:
- What problem this solves
- Who uses it and how
- Edge cases and constraints
- How it connects to existing code
- What "done" looks like

## When Building

Reference `docs/APPROACH.md` for patterns. Key points:

- Organize by feature, not by file type
- Use interface-implementation pattern for flexibility
- Add layers (repository, service) only when complexity demands it
- Types first, then implementation

## Process

1. Confirm you understand the feature
2. Ask your clarifying questions
3. Propose an approach (simple first)
4. Build incrementally, checking in as you go

Don't over-engineer. Start with the simplest thing that works.
