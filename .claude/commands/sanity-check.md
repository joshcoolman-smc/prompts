---
description: Critical code review and codebase health assessment
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

# Sanity Check

Perform a critical code review of the codebase and document findings.

## Instructions

### 1. Detect Project Type

Identify the project stack by checking for config files:

```bash
ls package.json tsconfig.json next.config.* vite.config.* nuxt.config.* vue.config.* remix.config.* astro.config.* angular.json 2>/dev/null
```

Note the framework(s) in use for context-aware analysis.

### 2. Review Recent Activity

```bash
git log --oneline -20
git status
```

Scan for TODO/FIXME comments:

```bash
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.vue" . 2>/dev/null | head -30
```

### 3. Explore Codebase Health

Use the Explore agent to investigate:

**Code Quality**
- Dead/unused exports, components, or functions
- Inconsistent patterns (naming, file structure, state management)
- Overly complex functions (high cyclomatic complexity)
- Missing error handling in critical paths

**Type Safety**
- Any usage of `any` type
- Missing return types on exported functions
- Untyped API responses or external data

**Security Concerns**
- Hardcoded secrets, API keys, or credentials
- Unsanitized user input
- Exposed sensitive routes or endpoints
- Console.log statements with sensitive data

**Dependencies**
- Outdated or deprecated packages (check package.json)
- Unused dependencies
- Duplicate functionality from multiple packages

**Documentation Drift**
- README accuracy vs actual project state
- Outdated env.example or .env.example
- Stale comments that no longer match code

### 4. Framework-Specific Checks

**If React/Next/Vite detected:**
- Missing key props in lists
- useEffect with missing deps or no cleanup
- State updates in render
- Large component files (>300 lines)

**If Vue/Nuxt detected:**
- Improper reactivity (mutating props, missing refs)
- Large single-file components
- Mixed Options/Composition API without pattern

**If TypeScript detected:**
- Strict mode enabled?
- Consistent use of interfaces vs types

### 5. Build Verification

Run production build and capture any errors:

```bash
npm run build 2>&1 || yarn build 2>&1 || pnpm build 2>&1
```

Check for type errors separately if applicable:

```bash
npx tsc --noEmit 2>&1 | head -50
```

### 6. Cruft Detection

Identify deletable files and dependencies. Build a list of candidates.

**Unused Dependencies**

Check for packages in package.json not imported anywhere:

```bash
# If npx depcheck available, use it
npx depcheck --json 2>/dev/null | head -100
```

If depcheck unavailable, manually scan: for each dependency in package.json, grep for its import. Flag any with zero matches.

**Orphaned Files**

Use Explore agent to find:

- Components/modules not imported by any other file
- Test files for components that no longer exist
- Storybook stories (.stories.tsx) for deleted components
- Config files for removed tools (e.g., .eslintrc for project using biome)
- Empty or near-empty files (<5 lines of actual code)
- Duplicate files (same content, different names/locations)

**Dead Code Patterns**

- Exports not imported anywhere else in codebase
- Functions defined but never called
- Commented-out code blocks (>10 lines)
- Feature flags that are always true/false
- ENV variables defined but never read

**Common Cruft Locations**

```bash
# Old build artifacts
ls -la dist/ build/ .next/ out/ 2>/dev/null

# Backup files
find . -name "*.bak" -o -name "*.old" -o -name "*~" -o -name "*.orig" 2>/dev/null

# OS/editor cruft
find . -name ".DS_Store" -o -name "Thumbs.db" -o -name "*.swp" 2>/dev/null
```

**Present Cruft for Cleanup**

If cruft found, present categorized list to user:

```
Question: "Found deletable cruft. What would you like to clean up?"
Header: "Cleanup"
Options:
  1. "Delete all safe cruft" (OS files, backups, empty files)
  2. "Review each category" (step through deps, orphans, dead code)
  3. "Skip cleanup" (just document in assessment)
```

**If "Delete all safe cruft":**
- Remove .DS_Store, *.bak, *.old, empty files
- Do NOT auto-delete dependencies or source files

**If "Review each category":**

For unused dependencies:
```
Question: "These dependencies appear unused: <list>. Remove from package.json?"
Header: "Unused deps"
Options:
  1. "Remove all listed"
  2. "Let me pick" (then list each with yes/no)
  3. "Keep all"
```

For orphaned files:
```
Question: "These files have no imports: <list max 10>. Delete?"
Header: "Orphans"
Options:
  1. "Delete all orphaned files"
  2. "Let me pick"
  3. "Keep all"
```

For dead exports/functions:
```
Question: "These exports are never imported: <list>. Remove?"
Header: "Dead code"
Options:
  1. "Remove dead exports"
  2. "Let me pick"
  3. "Keep all"
```

**After cleanup, note what was deleted in ASSESSMENT.md under a "Cleanup Performed" section.**

### 7. Write Findings to ASSESSMENT.md

Create or update `ASSESSMENT.md` at repo root:

```markdown
# Codebase Assessment

> Generated: <current-date>
> Stack: <detected-frameworks>

## Health Summary

**Overall: [Good | Needs Attention | Concerning]**

<1-2 sentence summary>

## Issues Found

### Critical
<issues that could cause bugs, security problems, or data loss>

### Major
<issues that hurt maintainability or developer experience>

### Minor
<nitpicks, style issues, cleanup opportunities>

## Cruft Identified

### Unused Dependencies
<list packages not imported anywhere>

### Orphaned Files
<files with no imports pointing to them>

### Dead Code
<exports/functions never used>

## Cleanup Performed

<if user chose to delete, list what was removed>

- Removed X unused dependencies
- Deleted Y orphaned files
- Cleaned Z dead exports

## Build Status

<build result and any warnings>

## Positive Findings

<what's working well, good patterns observed>

## Recommendations

<prioritized action items>

---

*Previous assessments archived below*
```

If `ASSESSMENT.md` already exists, move previous content under a dated archive section.

### 8. Commit the Assessment

Ask user before committing:

```
Question: "Commit the assessment?"
Header: "Commit"
Options:
  1. "Yes, commit ASSESSMENT.md"
  2. "Skip commit"
```

If yes:

```bash
git add ASSESSMENT.md
git commit -m "Update ASSESSMENT.md from sanity check"
git push
```

### 9. Verbal Summary

Conclude with:

```
Sanity Check Complete

Issues: X critical, Y major, Z minor
Cruft: X unused deps, Y orphaned files, Z dead exports
Cleaned: <what was deleted, if any>
Build: [Pass | Fail with N errors]
Health: [Good | Needs Attention | Concerning]

Top concerns:
1. <most urgent>
2. <second>
3. <third>
```

## Behavior

- Adapts checks to detected project type
- Interactive cleanup: identifies cruft, asks before deleting
- Safe defaults: auto-delete only OS/backup files, not source code
- Documents everything in ASSESSMENT.md before and after cleanup
- Asks before committing
- Focuses on actionable findings, not style nitpicks

## Error Handling

- If not in a git repo: Note this but continue analysis
- If no package.json: Treat as non-Node project, skip build checks
- If build script missing: Skip build verification, note in findings
