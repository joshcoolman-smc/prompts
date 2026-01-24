# React Feature Module Pattern

### Overview

Feature modules are self-contained units that:
- Group all related code together (components, hooks, services, repositories, types)
- Use repository pattern for data access abstraction
- Implement interface-first design for testability
- Use Zod for runtime validation at system boundaries
- Export a single display component as the public API

### File Structure

```
src/features/[feature-name]/
  /components/
    [FeatureName]Display.tsx      # Main composition component (exported)
    [Component1].tsx               # Feature-specific components
    [Component2].tsx               # (not shared across features)
  /hooks/
    use[FeatureName].ts            # Main feature hook
    use[Specific].ts               # Other custom hooks
  /repositories/
    [FeatureName]Repository.ts     # Interface definition
    [FeatureName]RepositoryImpl.ts # Implementation (API/DB/localStorage)
  /services/
    [FeatureName]Service.ts        # Business logic (optional - only if needed)
  /types/
    [feature-name].types.ts        # TypeScript interfaces/types
    [feature-name].schema.ts       # Zod schemas for validation
  /lib/
    [utility].ts                   # Feature-specific utilities (optional)
  index.ts                         # Public API - exports Display component
```

### Architecture Layers

**Flow:** Component → Hook → Service (optional) → Repository

#### 1. Types & Schemas (`/types`)

**TypeScript Types** (`[feature-name].types.ts`):
```typescript
export interface FeatureData {
  id: string
  name: string
  createdAt: Date
}

export interface FeatureFilters {
  search?: string
  status?: 'active' | 'inactive'
}
```

**Zod Schemas** (`[feature-name].schema.ts`):
```typescript
import { z } from 'zod'

export const featureDataSchema = z.object({
  id: z.string(),
  name: z.string(),
  createdAt: z.coerce.date()
})

export const featureFiltersSchema = z.object({
  search: z.string().optional(),
  status: z.enum(['active', 'inactive']).optional()
})
```

Use Zod schemas for:
- API response validation
- Form input validation
- External data validation
- Type inference: `type FeatureData = z.infer<typeof featureDataSchema>`

#### 2. Repository Pattern (`/repositories`)

**Interface** (`[FeatureName]Repository.ts`):
```typescript
import { FeatureData, FeatureFilters } from '../types/[feature-name].types'

export interface FeatureNameRepository {
  getAll(filters?: FeatureFilters): Promise<FeatureData[]>
  getById(id: string): Promise<FeatureData | null>
  create(data: Omit<FeatureData, 'id'>): Promise<FeatureData>
  update(id: string, data: Partial<FeatureData>): Promise<FeatureData>
  delete(id: string): Promise<void>
}
```

**Implementation** (`[FeatureName]RepositoryImpl.ts`):
```typescript
import { featureDataSchema } from '../types/[feature-name].schema'

export class FeatureNameRepositoryImpl implements FeatureNameRepository {
  constructor(private baseUrl: string) {}

  async getAll(filters?: FeatureFilters): Promise<FeatureData[]> {
    const response = await fetch(`${this.baseUrl}/items`)
    const json = await response.json()

    // Validate with Zod at the boundary
    return z.array(featureDataSchema).parse(json)
  }

  // ... other methods
}
```

**Key Principles:**
- Interface defines the contract, class implements it
- No business logic in repositories (just data access)
- Validate all external data with Zod schemas
- Easy to swap implementations (API → Mock → LocalStorage)

#### 3. Services (`/services`) - Optional

Only create services if you have business logic beyond simple CRUD:

```typescript
import { FeatureNameRepository } from '../repositories/[FeatureName]Repository'

export class FeatureNameService {
  constructor(private repository: FeatureNameRepository) {}

  async processFeature(id: string): Promise<void> {
    const data = await this.repository.getById(id)
    if (!data) throw new Error('Not found')

    // Business logic, validation, orchestration
    const processed = this.transform(data)
    await this.repository.update(id, processed)
  }

  private transform(data: FeatureData): Partial<FeatureData> {
    // Complex business rules here
    return { ...data }
  }
}
```

**When to use services:**
- Complex validation rules
- Multi-step workflows
- Coordinating multiple repositories
- Business logic that doesn't belong in UI

**When to skip services:**
- Simple CRUD operations (hook can call repository directly)
- No business logic beyond data fetching

#### 4. Custom Hooks (`/hooks`)

Connect React components to services/repositories:

```typescript
import { useState, useEffect } from 'react'
import { FeatureNameRepository } from '../repositories/[FeatureName]Repository'

interface UseFeatureNameProps {
  repository: FeatureNameRepository  // Injected for testability
}

export function useFeatureName({ repository }: UseFeatureNameProps) {
  const [data, setData] = useState<FeatureData[]>([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)

  const loadData = async () => {
    setLoading(true)
    setError(null)
    try {
      const result = await repository.getAll()
      setData(result)
    } catch (err) {
      setError(err as Error)
    } finally {
      setLoading(false)
    }
  }

  useEffect(() => {
    loadData()
  }, [])

  return { data, loading, error, reload: loadData }
}
```

**Key Principles:**
- Repository/service injected as parameter (not created inside hook)
- Manage loading, error, data states
- Return methods for mutations (create, update, delete)
- Keep UI concerns in hooks, not in services

#### 5. Components (`/components`)

**Display Component** (`[FeatureName]Display.tsx`):

The main composition component - this is what gets exported:

```typescript
import { Component1 } from './Component1'
import { Component2 } from './Component2'
import { useFeatureName } from '../hooks/use[FeatureName]'
import { FeatureNameRepositoryImpl } from '../repositories/[FeatureName]RepositoryImpl'

// Create repository instance
const repository = new FeatureNameRepositoryImpl('/api')

export function FeatureNameDisplay() {
  const { data, loading, error } = useFeatureName({ repository })

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>

  return (
    <div>
      <Component1 items={data} />
      <Component2 onAction={...} />
    </div>
  )
}
```

**Feature-Specific Components:**
- All components live in `/components` folder
- These are NOT shared across features
- Keep components focused and single-purpose
- Pass data and callbacks as props (no direct data fetching in components)

#### 6. Public API (`index.ts`)

Export only what other parts of the app need:

```typescript
// Export the display component (primary export)
export { FeatureNameDisplay } from './components/[FeatureName]Display'

// Export types if needed by other features
export type { FeatureData } from './types/[feature-name].types'

// Generally don't export: repositories, services, hooks, internal components
```

### Development Workflow

1. **Define Types First**
   - Create TypeScript interfaces in `/types/[feature-name].types.ts`
   - Create Zod schemas in `/types/[feature-name].schema.ts`

2. **Build Repository Interface**
   - Define contract in `[FeatureName]Repository.ts`
   - Implement in `[FeatureName]RepositoryImpl.ts`
   - Validate external data with Zod schemas

3. **Add Service Layer (if needed)**
   - Only if you have business logic beyond CRUD
   - Service takes repository as constructor parameter

4. **Create Custom Hook**
   - Takes repository/service as parameter
   - Manages React state (loading, error, data)
   - Returns data and methods for UI

5. **Build Components**
   - Start with Display component (composition)
   - Break down into smaller components as needed
   - Keep all components in `/components` folder

6. **Export Display Component**
   - Update `index.ts` to export Display component
   - This is the public API for the feature

### Key Principles

**Interface-Implementation Pattern:**
- Always define interface first, then implementation
- Makes testing easy (mock the interface)
- Allows swapping implementations without changing consumers

**Dependency Injection:**
- Pass dependencies as parameters (repository to hook, service to hook)
- Don't create instances inside hooks/components
- Create instances in Display component and pass down

**Validation at Boundaries:**
- Use Zod schemas for all external data (API responses, user input)
- Infer TypeScript types from Zod schemas when possible
- Internal data can use plain TypeScript types

**Feature Isolation:**
- Each feature is self-contained
- Components are NOT shared between features
- Shared components are a separate concern (live outside `/features`)

**Simplicity:**
- Only add layers when complexity demands it
- Simple CRUD? Skip the service layer
- No business logic? Hook calls repository directly
- Don't over-engineer

### Testing Considerations

The interface-implementation pattern makes testing straightforward:

```typescript
// Mock repository for testing
class MockFeatureRepository implements FeatureNameRepository {
  async getAll() { return [mockData1, mockData2] }
  // ... other methods return test data
}

// Test the hook
const { result } = renderHook(() =>
  useFeatureName({ repository: new MockFeatureRepository() })
)
```

### Integration Example

**In your router:**
```typescript
import { FeatureNameDisplay } from '@/features/feature-name'

<Route path="/feature" element={<FeatureNameDisplay />} />
```

**That's it.** The Display component handles everything internally.

### Common Patterns

**Multiple Data Sources:**
```typescript
// Define repository interface
export interface FeatureRepository {
  getAll(): Promise<Data[]>
}

// API implementation
export class ApiFeatureRepository implements FeatureRepository { ... }

// LocalStorage implementation
export class LocalFeatureRepository implements FeatureRepository { ... }

// Swap based on environment
const repository = import.meta.env.PROD
  ? new ApiFeatureRepository('/api')
  : new LocalFeatureRepository()
```

**Form Validation:**
```typescript
// Use Zod schema for form validation
const formSchema = z.object({
  name: z.string().min(1),
  email: z.string().email()
})

// In component
const handleSubmit = (formData: unknown) => {
  const validated = formSchema.parse(formData)
  repository.create(validated)
}
```

**Error Handling:**
```typescript
// Repository throws errors
async create(data: Data): Promise<Data> {
  const response = await fetch(...)
  if (!response.ok) throw new Error('Failed to create')
  return featureDataSchema.parse(await response.json())
}

// Hook catches errors
try {
  await repository.create(data)
} catch (err) {
  setError(err as Error)
}

// Component displays errors
{error && <div className="error">{error.message}</div>}
```

### What NOT to Do

- Don't create shared component folders at the feature level
- Don't skip the interface (always define contract first)
- Don't put business logic in repositories (data access only)
- Don't create services if you just need simple CRUD
- Don't create instances inside hooks (inject dependencies)
- Don't skip Zod validation at system boundaries
- Don't export internal components from `index.ts`

### Verification Checklist

- [ ] Types defined with TypeScript interfaces
- [ ] Zod schemas for external data validation
- [ ] Repository interface defined
- [ ] Repository implementation validates with Zod
- [ ] Service layer only if business logic exists
- [ ] Custom hook takes repository/service as parameter
- [ ] Display component is the composition root
- [ ] All feature components in `/components` folder
- [ ] Only Display component exported from `index.ts`
- [ ] Dependencies injected (not created inside hooks)

---

**End of Prompt**

Use this pattern for consistent, testable, maintainable feature development in React applications.
