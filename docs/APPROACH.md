# Approach

How we build things. Framework-agnostic, stack-flexible.

## Philosophy

**Pragmatism over dogma.** Principles that flex, not rigid processes.

**Minimal tooling.** Editor, terminal, AI assistant, markdown files. No wrappers on wrappers.

**Close to core tools.** Work with Claude Code/Gemini CLI directly, not abstractions on top.

**Add complexity only when needed.** Three similar lines > premature abstraction.

## Organizing Code

**By feature, not by type.** Group everything related to a feature together:

```
features/
  user-profile/
    components/
    hooks/
    services/
    repositories/
    types/
```

Not scattered `components/`, `services/`, `hooks/` folders at the root.

## Three-Layer Pattern

When you need structure beyond simple components:

**Repository** (Data Access)
- Abstracts where data comes from (API, database, local storage)
- Interface defines the contract, class implements it
- No business logic here

**Service** (Business Logic)
- Orchestrates rules, coordinates repositories
- Where validation, transformation, workflows live
- Only add if there's logic beyond simple CRUD

**Hooks** (UI Integration)
- Connect React/Vue/whatever to services
- Manage loading, error, data state
- Services injected as parameters (testability)

Flow: Components → Hooks → Services → Repositories

## Interface-Implementation Pattern

Define what can be done separately from how it's done:

```typescript
interface PaymentProcessor {
  charge(amount: number): Promise<Result>
}

class StripeProcessor implements PaymentProcessor { ... }
class MockProcessor implements PaymentProcessor { ... }
```

**Why:**
- Swap implementations without changing consuming code
- Easy to test with mocks
- Contract is explicit

**When:**
- Repositories: always (data sources vary)
- Services: when business logic has variants
- Utilities: usually don't need (pure functions)

## Type Safety

- TypeScript strict mode
- Zod for runtime validation at boundaries (API responses, form inputs)
- Define types close to their feature

## Testing

TDD when it makes sense. Tests define success criteria before implementation.

**But pragmatically:** Don't force TDD on trivial changes. Write tests for behavior, not implementation details.

## What We Don't Do

- Over-engineer for hypothetical futures
- Add features/refactoring beyond what was asked
- Create abstractions for one-time operations
- Layer tool upon tool
- Extensive upfront planning before knowing what we're building

## Starting a Feature

1. Clarify what we're building (ask questions)
2. Define types first
3. Build the simplest thing that works
4. Add layers (repository, service) only if complexity demands it
5. Test the important paths

---

*This is how we approach building. Specific tech choices live in the project, not here.*
