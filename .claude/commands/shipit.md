---
description: Ship completed work with retrospective and issue closure
argument-hint: [optional #issue-number]
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - AskUserQuestion
  - Glob
---

# Ship It

Complete and ship your work by adding a retrospective to the GitHub issue, closing it, updating project docs, and committing all changes.

## Instructions

When the user runs `/shipit` (with optional issue number like `/shipit #42`):

### 1. Determine Issue Number

**If issue number provided as argument:**

- Use that issue number directly

**If no argument provided:**

- Check conversation context for the issue being worked on
- If unclear, ask: "Which issue should I close?" and list recent open issues:
  ```bash
  gh issue list --limit 10
  ```

### 2. Add Retrospective Comment

Create a retrospective comment on the issue:

```bash
gh issue comment <issue-number> --body "$(cat <<'EOF'
<retrospective content>
EOF
)"
```

The retrospective should include:

- **Key Learnings**: Technical insights, decisions made
- **Implementation Details**: High-level summary of changes
- **Potential Future Improvements**: Ideas for enhancement
- **Testing Notes**: What was verified

### 2.5. Check for Feature Documentation

After adding the retrospective comment, check if feature documentation should be updated:

1. **Check if `docs/features/` exists:**

   ```bash
   ls docs/features/*.md 2>/dev/null
   ```

2. **If it exists and has Markdown files**, ask the user:

   Use AskUserQuestion to present options based on existing feature docs:

   ```
   Question: "Would you like to update feature documentation?"
   Header: "Feature docs"
   Options:
     1. "Update: <existing-file>.md" (list existing .md files, excluding _TEMPLATE.md)
     2. "Create new feature doc"
     3. "Skip"
   ```

3. **If user chooses to update/create**:
   - **Update**: Read the existing doc, add relevant updates to appropriate sections
   - **Create**: Copy from `_TEMPLATE.md`, ask for feature name, populate basics
   - Note what was updated in confirmation output

4. **If `docs/features/` doesn't exist**, ask the user:

   Use AskUserQuestion:

   ```
   Question: "No docs folder found. Would you like to bootstrap documentation for this repo?"
   Header: "Bootstrap docs"
   Options:
     1. "Yes, create docs structure"
     2. "Skip"
   ```

   **If user chooses to bootstrap:**
   - Create `docs/features/_TEMPLATE.md` with standard feature doc template
   - Create `docs/01-progress.md` with initial structure
   - Create a feature doc for the current work being shipped
   - Note in confirmation: "Bootstrapped docs/ structure"

### 3. Close Issue

Close the GitHub issue with a completion note:

```bash
gh issue close <issue-number> --comment "Implementation completed and verified."
```

### 4. Update Progress Log (if exists)

Check if `docs/01-progress.md` or `docs/progress.md` exists in the project:

```bash
ls docs/01-progress.md docs/progress.md 2>/dev/null
```

**If found**, add an entry at the top (after the first `---` separator):

- `###` heading with descriptive title
- `**Issue:** #<number> - <issue title>`
- Brief summary paragraph of what was implemented
- `**Key files:**` list of main files changed
- `---` separator

### 5. Update README (if significant)

Check if `README.md` exists and if the work warrants an update:

**Update README if the work:**

- Added a new route/page
- Added a new major feature
- Changed the tech stack
- Modified the project structure
- Added new commands or workflows

**Skip README update if:**

- Bug fix only
- Minor refactor
- Documentation-only changes
- Internal implementation details

If updating, keep changes minimal - just add/modify the relevant section. Don't rewrite the whole README.

If unsure whether to update, ask: "This added [feature]. Should I update the README?"

### 6. Commit and Push

Stage all changes:

```bash
git add -A
```

Create a commit with a descriptive message that:

- Summarizes the changes
- References the issue number: "Fixes #<number>"
- Includes `Co-Authored-By: Claude <noreply@anthropic.com>`

Push to the current branch:

```bash
git push
```

### 7. Confirm to User

Display a confirmation message:

```
Shipped!

- Issue #<number> closed with retrospective
- Feature docs updated: <feature-name>.md (if applicable)
- Progress log updated (if applicable)
- README updated (if applicable)
- Changes committed and pushed

Anything else?
```

## Behavior

- **Only triggers with `/shipit` command** (not from natural language)
- **Context-aware**: Uses conversation history to infer issue number
- **Explicit override**: Can specify issue with `/shipit #42`
- **Fallback**: Asks user if issue can't be determined
- **Project-aware**: Adapts to project structure (progress.md, README)

## Error Handling

- If `gh` CLI not installed: Exit with message to install GitHub CLI
- If `gh` not authenticated: Exit with message to run `gh auth login`
- If not in a git repo: Exit with helpful message
- If git operations fail: Report the error clearly

## Examples

```bash
# Ship current work (infers issue from context)
/shipit

# Ship specific issue
/shipit #42
```

## Notes

- This command only triggers via explicit `/shipit` slash command
- Does not auto-trigger from phrases like "ship it" or "commit"
- Works in any GitHub repository
- No workflow state files needed
- Progress log and README updates only happen if those files exist
