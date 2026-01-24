# Docs Feature Prompt

This prompt can be used to reliably reproduce the documentation feature in any React/Vite application.

---

## Prompt: Drop-In Documentation Feature for React/Vite Apps

I want to add a self-contained documentation system to my React/Vite application. Here are the requirements:

### Overview
Create a documentation feature where:
- Documentation files are Markdown (`.md`) stored in `/docs` folder at project root
- Docs are loaded at build time using Vite's `import.meta.glob` (zero backend)
- A dedicated `/docs` route displays documentation with sidebar navigation
- Document titles are automatically derived from filenames (e.g., `getting-started.md` → "Getting Started")
- Features syntax highlighting for code blocks and professional typography
- Everything is self-contained in `/src/features/docs/` for portability

### Technology Stack
- React Router for navigation
- `react-markdown` for rendering Markdown
- `remark-gfm` for GitHub-flavored Markdown support
- `rehype-sanitize` for XSS protection
- `react-syntax-highlighter` with VS Code Dark+ theme for code highlighting
- `@tailwindcss/typography` for beautiful prose styles

### File Structure

Create this structure:

```
/docs/                              # Markdown files (project root)
  getting-started.md
  [other docs].md

/src/features/docs/
  /components/
    DocsPage.tsx                    # Main route component with routing logic
    DocLayout.tsx                   # Two-column layout (sidebar + content)
    DocSidebar.tsx                  # Navigation sidebar with doc list
    DocContent.tsx                  # Markdown renderer with syntax highlighting
  /hooks/
    useDocs.ts                      # Hook to load and manage docs
  /lib/
    loadDocs.ts                     # Vite import.meta.glob logic
    parseDocTitle.ts                # Convert filename to title
    slugify.ts                      # Convert title to URL slug
  /types/
    docs.ts                         # TypeScript interfaces
  config.ts                         # Configuration (sidebar width, default doc)
  index.ts                          # Public API exports
```

### Implementation Details

#### 1. Install Dependencies
```bash
npm install react-router-dom react-markdown remark-gfm rehype-sanitize react-syntax-highlighter @types/react-syntax-highlighter @tailwindcss/typography
```

#### 2. TypeScript Types (`src/features/docs/types/docs.ts`)
```typescript
export interface DocFile {
  slug: string
  title: string
  content: string
}

export interface DocMetadata {
  slug: string
  title: string
}
```

#### 3. Utilities

**Slugify** (`src/features/docs/lib/slugify.ts`):
- Convert text to lowercase
- Replace spaces with hyphens
- Remove special characters
- Clean up multiple/leading/trailing hyphens

**Parse Doc Title** (`src/features/docs/lib/parseDocTitle.ts`):
- Remove `.md` extension from filename
- Split on hyphens/underscores
- Capitalize each word
- Join with spaces

**Load Docs** (`src/features/docs/lib/loadDocs.ts`):
- Use `import.meta.glob("/docs/*.md", { query: "?raw", import: "default" })`
- Export `loadAllDocs()` that returns `Promise<DocFile[]>`
- Export `loadDocMetadata()` that returns `Promise<DocMetadata[]>`
- Sort docs alphabetically by title

#### 4. Hook (`src/features/docs/hooks/useDocs.ts`)
Create a hook that:
- Loads docs on mount using `loadAllDocs()` and `loadDocMetadata()`
- Returns `{ docs, metadata, loading, error }`
- Handles loading states and errors

#### 5. Components

**DocLayout** (`src/features/docs/components/DocLayout.tsx`):
- Two-column flexbox layout
- Accept `sidebar` and `content` as props
- Full height (`h-screen`)

**DocSidebar** (`src/features/docs/components/DocSidebar.tsx`):
- Fixed width (280px), dark background
- Map over `metadata` to render navigation links
- Use React Router `<Link>` components
- Highlight active doc based on URL param
- Include "Back to App" link at bottom

**DocContent** (`src/features/docs/components/DocContent.tsx`):
- Main content area, scrollable
- Display doc title as h1
- Wrap ReactMarkdown in `prose prose-invert prose-zinc max-w-none` classes
- Use `remarkGfm` and `rehypeSanitize` plugins
- Override `code` component to use `react-syntax-highlighter`:
  - Inline code: render as simple `<code>` tag
  - Code blocks: use `SyntaxHighlighter` with VS Code Dark+ theme
  - Extract language from className (`language-xxx`)
  - Use `not-prose` class on SyntaxHighlighter to prevent conflicts
- Override `pre` component to return children directly (avoid double wrapping)

**DocsPage** (`src/features/docs/components/DocsPage.tsx`):
- Use `useParams<{ slug?: string }>()` to get current doc slug
- Use `useDocs()` hook
- Show loading state while docs load
- Show error state if loading fails
- Redirect to default doc if no slug provided
- Redirect to default doc if slug not found
- Render `DocLayout` with `DocSidebar` and `DocContent`

#### 6. Configuration (`src/features/docs/config.ts`)
```typescript
export const docsConfig = {
  sidebarWidth: "280px",
  defaultDoc: "getting-started", // slug of default doc
}
```

#### 7. Public API (`src/features/docs/index.ts`)
Export `DocsPage` component and types.

#### 8. Integration

**Add Tailwind Typography Plugin**:
In your CSS file (e.g., `src/index.css`), add:
```css
@plugin "@tailwindcss/typography";
```

**Set Up Routing**:
In `src/main.tsx`, wrap your app with React Router:
```typescript
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import { DocsPage } from './features/docs'

<BrowserRouter>
  <Routes>
    <Route path="/" element={<YourMainApp />} />
    <Route path="/docs" element={<DocsPage />} />
    <Route path="/docs/:slug" element={<DocsPage />} />
  </Routes>
</BrowserRouter>
```

**Add Link to Docs**:
In your main app UI (sidebar, header, etc.), add a link to `/docs`:
```typescript
import { Link } from 'react-router-dom'

<Link to="/docs">Docs</Link>
```

### Styling Guidelines
- Use Tailwind utility classes throughout
- Dark theme: zinc color palette (zinc-900, zinc-800, zinc-700, etc.)
- Sidebar: dark background with hover states on links
- Active link: lighter background and bold text
- Content area: max-width for readability (max-w-4xl)
- Code blocks: rounded corners, VS Code Dark+ theme
- Inline code: rounded background, monospace font

### Initial Documentation
Create at least one initial doc in `/docs/getting-started.md` with basic setup instructions.

### Key Implementation Notes

**Vite Import Path**:
Use `/docs/*.md` (absolute from project root) for `import.meta.glob`.

**Routing Pattern**:
Use explicit routes (`/docs` and `/docs/:slug`) NOT catch-all (`/docs/*`) for proper React Router behavior.

**Syntax Highlighting**:
- Import from `react-syntax-highlighter/dist/esm/styles/prism`
- Use `vscDarkPlus` theme
- Apply `not-prose` class to prevent typography conflicts
- Handle both inline code and code blocks differently

**Typography Integration**:
- Apply `prose prose-invert prose-zinc max-w-none` to markdown container
- This handles most styling automatically
- Only override components where custom behavior is needed (code blocks)

**Error Handling**:
- Handle loading states
- Handle empty docs array
- Handle missing slugs (redirect to default)
- Handle doc not found (redirect to default)

### Verification Steps
1. Build should succeed: `npm run build`
2. Navigate to `/docs` - should redirect to default doc
3. Sidebar shows all docs from `/docs/` folder
4. Clicking a doc updates URL and content
5. Browser back/forward buttons work
6. Code blocks have syntax highlighting
7. Typography looks polished and professional
8. Add a new `.md` file to `/docs/` and rebuild - should appear in sidebar

### Portability Notes
This feature is designed to be portable:
- Copy `/src/features/docs/` to any React/Vite project
- Copy `/docs/` folder with documentation
- Install dependencies
- Set up routing and add link
- Requires: React 18+, Vite, React Router, Tailwind CSS

---

**End of Prompt**

Copy everything above this line and paste into an LLM to reproduce this documentation feature.
