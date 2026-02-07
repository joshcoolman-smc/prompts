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

### 6. Write Findings to ASSESSMENT.md

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

### 7. Commit the Assessment

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

### 8. Verbal Summary

Conclude with:

```
Sanity Check Complete

Issues: X critical, Y major, Z minor
Build: [Pass | Fail with N errors]
Health: [Good | Needs Attention | Concerning]

Top concerns:
1. <most urgent>
2. <second>
3. <third>
```

## Behavior

- Adapts checks to detected project type
- Non-destructive (only writes ASSESSMENT.md)
- Asks before committing
- Focuses on actionable findings, not style nitpicks

## Error Handling

- If not in a git repo: Note this but continue analysis
- If no package.json: Treat as non-Node project, skip build checks
- If build script missing: Skip build verification, note in findings
