---
name: whats-up
description: Quick orientation for returning to a conversation. Use when the user says "what's up?" to get a summary of the current repo, what's being worked on, and the last file edited.
user_invocable: true
invocation_hint: what's up
---

# What's Up

Provide a quick orientation when the user returns to a conversation.

## Instructions

When invoked, respond with exactly three lines:

1. **Repo** -- Run `basename $(git rev-parse --show-toplevel 2>/dev/null) 2>/dev/null || basename $(pwd)` to get the repo/directory name. Report it.
2. **Working on** -- Scan the conversation context and summarize in one sentence what we've been doing. If this is a fresh conversation with no prior context, say "Fresh session -- nothing in progress."
3. **Last file** -- If you edited or read a file earlier in this conversation, show its path. If none, say "No files touched yet."

## Output format

```
Repo: <repo-name>
Working on: <one sentence>
Last file: <path or "No files touched yet">
```

Keep it to just these three lines. No extra commentary.
