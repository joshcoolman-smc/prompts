# AI-First Development Starter

A starting point for AI-assisted development.

Just a small set of files that establish:

- **How we build** - Patterns and philosophy
- **Where we are** - Session continuity
- **How to start** - Commands that ask questions before coding

## Structure

```
docs/
  APPROACH.md     # Patterns, philosophy, how we build things
  CURRENT.md      # What's happening now, session handoff

.claude/
  commands/
    feature.md    # Start a new feature
    review.md     # Review code
    design.md     # Design direction

CLAUDE.md         # Entry point for AI assistants
```

## The Idea

Just markdown files that establish working patterns. No magic, no ceremony.

## Using It

1. Copy these files into your project
2. Fill in `docs/CURRENT.md` with your current sprint/goals
3. Adjust `docs/APPROACH.md` if your philosophy differs
4. Use `/feature`, `/review`, `/design` as starting points

## Philosophy

- Pragmatism over dogma
- Minimal tooling
- Ask before building
- Add complexity only when needed
- Framework-agnostic (works with React, Vue, CLI tools, whatever)

## Commands

- `/feature [description]` - Start a new feature with clarifying questions
- `/review [file]` - Code review with actionable feedback
- `/design [request]` - Design system direction

---

_For weekly sessions with collaborators: keep `docs/CURRENT.md` updated so the next session can pick up where you left off._
