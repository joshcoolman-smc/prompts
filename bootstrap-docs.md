# Bootstrap Docs Viewer

Drop-in docs viewer for any TypeScript/React/Tailwind project. Point an AI agent at this file and it will create a fully working `/docs` route with sidebar navigation, syntax highlighting, and table of contents.

## How to use

1. Initialize your project (Next.js, Vite, TanStack Start, etc.)
2. Place this file in the project root (or reference it)
3. Tell the agent: "Read bootstrap-docs.md and follow the instructions"

## What gets created

```
project/
├── docs/
│   ├── 01-progress.md
│   └── features/
│       └── _TEMPLATE.md
├── .claude/
│   └── commands/
│       └── shipit.md
├── lib/docs/                   # (or src/lib/docs/)
│   ├── types.ts
│   ├── config.ts
│   ├── slugify.ts
│   ├── parseFileName.ts
│   ├── parseDocTitle.ts
│   ├── extractHeadings.ts
│   └── loadDocs.ts
├── [route]/docs/               # Framework-specific routes
│   ├── layout
│   ├── page (index/redirect)
│   └── [...slug] (catch-all)
└── [components]/docs/
    ├── DocsSidebar
    ├── TableOfContents
    ├── CodeBlock
    └── DocsContent
```

---

## Step 1: Detect Project Structure

Read `package.json` and determine:

1. **Framework**: Look for `next`, `vite`, `@tanstack/start`, `remix`, `astro` in dependencies/devDependencies
2. **Source dir**: Check if `src/` directory exists. If yes, utilities go in `src/lib/docs/`. If no, `lib/docs/`.
3. **Tailwind version**: Check for `tailwindcss` version. v4 uses `@import "tailwindcss"` in CSS. v3 uses `tailwind.config`.
4. **shadcn/ui**: Check for `components.json` or `@radix-ui/*` in dependencies. If present, sidebar color variables (`sidebar-*`) are available. If not, use standard Tailwind classes.
5. **Non-monorepo assumption**: Docs live at `process.cwd()/docs`.

Only ask the user if the framework cannot be determined.

---

## Step 2: Install Dependencies

```bash
# Core (always needed)
npm install gray-matter remark-gfm rehype-slug react-syntax-highlighter lucide-react
npm install -D @types/react-syntax-highlighter

# MDX - install ONE based on framework:
# Next.js App Router:
npm install next-mdx-remote
# Vite:
npm install @mdx-js/rollup @mdx-js/react
# Generic React:
npm install @mdx-js/react
```

---

## Step 3: Create Utility Files

All files go in `lib/docs/` (or `src/lib/docs/` if project uses `src/`).

### lib/docs/types.ts

```typescript
export interface DocMetadata {
  title: string;
  description?: string;
  order?: number;
  published?: boolean;
  category?: string | null;
}

export interface DocFile {
  slug: string;
  category: string | null;
  metadata: DocMetadata;
  content: string;
}

export interface DocNavItem {
  slug: string;
  title: string;
  order: number;
  category: string | null;
}

export interface DocNavCategory {
  name: string | null;
  displayName: string;
  items: DocNavItem[];
  order: number;
}
```

### lib/docs/config.ts

```typescript
import path from "path";

export const DOCS_CONFIG = {
  docsDirectory: path.join(process.cwd(), "docs"),
  defaultDoc: "getting-started",
  fileExtension: ".md",
} as const;
```

### lib/docs/slugify.ts

```typescript
export function slugify(text: string): string {
  return text
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, "")
    .replace(/[\s_-]+/g, "-")
    .replace(/^-+|-+$/g, "");
}

export function slugToTitle(slug: string): string {
  return slug
    .split("-")
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1))
    .join(" ");
}
```

### lib/docs/parseFileName.ts

```typescript
export function parseFileName(filename: string): {
  prefix: number | null;
  cleanName: string;
} {
  const match = filename.match(/^(\d+)-(.+)$/);

  if (match && match[1] && match[2]) {
    return {
      prefix: parseInt(match[1], 10),
      cleanName: match[2],
    };
  }

  return {
    prefix: null,
    cleanName: filename,
  };
}

export function sortFileNames(filenames: string[]): string[] {
  return filenames.sort((a, b) => {
    const parsedA = parseFileName(a);
    const parsedB = parseFileName(b);

    if (parsedA.prefix !== null && parsedB.prefix !== null) {
      return parsedA.prefix - parsedB.prefix;
    }

    if (parsedA.prefix !== null) return -1;
    if (parsedB.prefix !== null) return 1;

    return parsedA.cleanName.localeCompare(parsedB.cleanName);
  });
}
```

### lib/docs/parseDocTitle.ts

```typescript
export function parseDocTitle(content: string): string | null {
  const h1Match = content.match(/^#\s+(.+)$/m);
  return h1Match?.[1]?.trim() ?? null;
}
```

### lib/docs/extractHeadings.ts

```typescript
export interface TocHeading {
  id: string;
  text: string;
  depth: number;
}

function slugify(text: string): string {
  return text
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, "")
    .replace(/[\s]+/g, "-")
    .replace(/--+/g, "-");
}

export function extractHeadings(markdown: string): TocHeading[] {
  const headings: TocHeading[] = [];
  const lines = markdown.split("\n");

  for (const line of lines) {
    const match = line.match(/^(##)\s+(.+)$/);
    if (match && match[1] && match[2]) {
      const text = match[2].trim();
      headings.push({
        id: slugify(text),
        text,
        depth: 2,
      });
    }
  }

  return headings;
}
```

### lib/docs/loadDocs.ts

```typescript
import fs from "fs/promises";
import path from "path";
import matter from "gray-matter";
import { DOCS_CONFIG } from "./config";
import type {
  DocFile,
  DocMetadata,
  DocNavItem,
  DocNavCategory,
} from "./types";
import { slugToTitle } from "./slugify";
import { parseFileName, sortFileNames } from "./parseFileName";

interface DocEntry {
  category: string | null;
  fileName: string;
  slug: string;
  path: string;
}

async function scanDocsDirectory(): Promise<DocEntry[]> {
  const entries: DocEntry[] = [];
  const rootFiles = await fs.readdir(DOCS_CONFIG.docsDirectory, {
    withFileTypes: true,
  });

  for (const entry of rootFiles) {
    if (entry.isFile() && entry.name.endsWith(".md")) {
      if (entry.name.startsWith("_")) continue;

      entries.push({
        category: null,
        fileName: entry.name,
        slug: entry.name.replace(".md", ""),
        path: path.join(DOCS_CONFIG.docsDirectory, entry.name),
      });
    } else if (entry.isDirectory()) {
      const categoryName = entry.name;
      const subFiles = await fs.readdir(
        path.join(DOCS_CONFIG.docsDirectory, categoryName),
        { withFileTypes: true }
      );

      for (const subFile of subFiles) {
        if (subFile.isFile() && subFile.name.endsWith(".md")) {
          if (subFile.name.startsWith("_")) continue;

          entries.push({
            category: categoryName,
            fileName: subFile.name,
            slug: subFile.name.replace(".md", ""),
            path: path.join(DOCS_CONFIG.docsDirectory, categoryName, subFile.name),
          });
        }
      }
    }
  }

  return entries;
}

export async function getAllDocSlugs(): Promise<string[]> {
  try {
    const entries = await scanDocsDirectory();

    const slugMap = new Map<string, string>();
    const fullSlugs = entries.map((entry) => {
      const fullSlug = entry.category
        ? `${entry.category}/${entry.slug}`
        : entry.slug;
      if (slugMap.has(fullSlug)) {
        throw new Error(
          `Slug collision: "${fullSlug}" exists at:\n` +
            `  - ${slugMap.get(fullSlug)}\n` +
            `  - ${entry.path}`
        );
      }
      slugMap.set(fullSlug, entry.path);
      return fullSlug;
    });

    return fullSlugs;
  } catch (error) {
    console.error("Error reading docs directory:", error);
    return [];
  }
}

export async function getDocBySlug(slug: string): Promise<DocFile | null> {
  try {
    const filePath = path.join(
      DOCS_CONFIG.docsDirectory,
      `${slug}${DOCS_CONFIG.fileExtension}`
    );
    const fileContent = await fs.readFile(filePath, "utf-8");
    const { data, content } = matter(fileContent);

    const parts = slug.split("/");
    const category: string | null = parts.length > 1 ? parts[0]! : null;
    const slugName = parts[parts.length - 1] ?? slug;

    const { cleanName } = parseFileName(slugName);
    const title = slugToTitle(cleanName);

    const metadata: DocMetadata = {
      title,
      description: data.description as string | undefined,
      order: data.order as number | undefined,
      published: data.published !== false,
      category,
    };

    return {
      slug,
      category,
      metadata,
      content,
    };
  } catch (error) {
    console.error(`Error loading doc "${slug}":`, error);
    return null;
  }
}

export async function getAllDocs(): Promise<DocFile[]> {
  const slugs = await getAllDocSlugs();
  const docs = await Promise.all(slugs.map((slug) => getDocBySlug(slug)));
  return docs.filter((doc): doc is DocFile => doc !== null);
}

export async function getDocNavCategories(): Promise<DocNavCategory[]> {
  const docs = await getAllDocs();
  const published = docs.filter((doc) => doc.metadata.published !== false);

  const categoryMap = new Map<string | null, DocNavItem[]>();

  for (const doc of published) {
    const category = doc.category;
    if (!categoryMap.has(category)) {
      categoryMap.set(category, []);
    }
    categoryMap.get(category)!.push({
      slug: doc.slug,
      title: doc.metadata.title,
      order: doc.metadata.order ?? 999,
      category: doc.category,
    });
  }

  const categories: DocNavCategory[] = [];

  for (const [categoryName, items] of categoryMap) {
    const sortedItems = items.sort((a, b) => {
      const aSlug = a.slug.split("/").pop()!;
      const bSlug = b.slug.split("/").pop()!;
      const parsedA = parseFileName(aSlug);
      const parsedB = parseFileName(bSlug);

      if (parsedA.prefix !== null && parsedB.prefix !== null) {
        return parsedA.prefix - parsedB.prefix;
      }
      if (parsedA.prefix !== null) return -1;
      if (parsedB.prefix !== null) return 1;
      return parsedA.cleanName.localeCompare(parsedB.cleanName);
    });

    categories.push({
      name: categoryName,
      displayName: categoryName
        ? categoryName.charAt(0).toUpperCase() + categoryName.slice(1)
        : "Documentation",
      items: sortedItems,
      order: categoryName === null ? 0 : 1,
    });
  }

  return categories.sort((a, b) => {
    if (a.order !== b.order) return a.order - b.order;
    return a.displayName.localeCompare(b.displayName);
  });
}
```

---

## Step 4: Create Components

Place in the framework's component directory (e.g., `app/(docs)/_components/` for Next.js, `src/components/docs/` for Vite).

### CodeBlock

```tsx
"use client"; // FRAMEWORK: Remove if not Next.js

import { Prism as SyntaxHighlighter } from "react-syntax-highlighter";
import { vscDarkPlus } from "react-syntax-highlighter/dist/esm/styles/prism";

export function CodeBlock({ className, children, ...props }: any) {
  const match = /language-(\w+)/.exec(className || "");
  const language = match ? match[1] : "";

  return match ? (
    <SyntaxHighlighter
      style={vscDarkPlus}
      language={language}
      PreTag="div"
      customStyle={{
        borderRadius: "0.75rem",
        padding: "1.5rem",
      }}
      {...props}
    >
      {String(children).replace(/\n$/, "")}
    </SyntaxHighlighter>
  ) : (
    <code className={className} {...props}>
      {children}
    </code>
  );
}
```

### TableOfContents

```tsx
"use client"; // FRAMEWORK: Remove if not Next.js

import { useEffect, useRef, useState } from "react";
// FRAMEWORK: Adjust import path to match your lib/docs location
import type { TocHeading } from "@/lib/docs/extractHeadings";

interface TableOfContentsProps {
  headings: TocHeading[];
}

export function TableOfContents({ headings }: TableOfContentsProps) {
  const [activeId, setActiveId] = useState<string>("");
  const observerRef = useRef<IntersectionObserver | null>(null);

  useEffect(() => {
    const headingElements = headings
      .map((h) => document.getElementById(h.id))
      .filter(Boolean) as HTMLElement[];

    if (headingElements.length === 0) return;

    observerRef.current = new IntersectionObserver(
      (entries) => {
        const visibleEntries = entries.filter((e) => e.isIntersecting);
        if (visibleEntries.length > 0 && visibleEntries[0]) {
          setActiveId(visibleEntries[0].target.id);
        }
      },
      {
        rootMargin: "-80px 0px -60% 0px",
        threshold: 0,
      }
    );

    headingElements.forEach((el) => observerRef.current?.observe(el));

    return () => {
      observerRef.current?.disconnect();
    };
  }, [headings]);

  const handleClick = (id: string) => {
    const el = document.getElementById(id);
    if (el) {
      el.scrollIntoView({ behavior: "smooth" });
    }
  };

  return (
    <div className="hidden xl:block sticky top-22 h-fit max-h-[calc(100vh-6rem)] overflow-y-auto w-56 shrink-0">
      <p className="text-sm font-semibold text-muted-foreground mb-3">
        On this page
      </p>
      <ul className="space-y-1">
        {headings.map((heading) => {
          const isActive = activeId === heading.id;

          return (
            <li key={heading.id}>
              <button
                onClick={() => handleClick(heading.id)}
                className={`block w-full text-left text-sm py-1 pl-3 transition-colors ${
                  isActive
                    ? "text-foreground border-l-2 border-foreground"
                    : "text-muted-foreground border-l border-border hover:text-foreground/80"
                }`}
              >
                {heading.text}
              </button>
            </li>
          );
        })}
      </ul>
    </div>
  );
}
```

### DocsSidebar

```tsx
"use client"; // FRAMEWORK: Remove if not Next.js

// FRAMEWORK: Next.js uses these imports. Adapt for your router:
//   Vite + React Router: import { Link, useLocation } from "react-router-dom"; (useLocation().pathname)
//   TanStack Router: import { Link, useLocation } from "@tanstack/react-router"; (useLocation().pathname)
import Link from "next/link";
import { usePathname } from "next/navigation";
import { useState, useEffect } from "react";
import { ChevronRight } from "lucide-react";
// FRAMEWORK: Adjust import path
import type { DocNavCategory } from "@/lib/docs/types";

interface DocsSidebarProps {
  categories: DocNavCategory[];
}

export function DocsSidebar({ categories }: DocsSidebarProps) {
  // FRAMEWORK: Next.js uses usePathname(). Other routers: useLocation().pathname
  const pathname = usePathname();

  const [collapsedCategories, setCollapsedCategories] = useState<Set<string>>(() => {
    if (typeof window === 'undefined') {
      return new Set(categories.filter(c => c.name !== null).map(c => c.name!));
    }
    const saved = localStorage.getItem('docs-sidebar-collapsed');
    return saved
      ? new Set(JSON.parse(saved))
      : new Set(categories.filter(c => c.name !== null).map(c => c.name!));
  });

  useEffect(() => {
    localStorage.setItem('docs-sidebar-collapsed', JSON.stringify([...collapsedCategories]));
  }, [collapsedCategories]);

  useEffect(() => {
    categories.forEach(category => {
      if (category.name && category.items.some(item => `/docs/${item.slug}` === pathname)) {
        setCollapsedCategories(prev => {
          const next = new Set(prev);
          next.delete(category.name!);
          return next;
        });
      }
    });
  }, [pathname, categories]);

  const toggleCategory = (categoryName: string) => {
    setCollapsedCategories(prev => {
      const next = new Set(prev);
      if (next.has(categoryName)) {
        next.delete(categoryName);
      } else {
        next.add(categoryName);
      }
      return next;
    });
  };

  return (
    <aside className="fixed left-0 top-0 w-64 h-screen border-r border-border bg-background overflow-y-auto">
      <nav className="p-6 space-y-6">
        {categories.map((category) => {
          const isCollapsed = category.name && collapsedCategories.has(category.name);

          return (
            <div key={category.name ?? "root"}>
              {category.name ? (
                <button
                  onClick={() => toggleCategory(category.name!)}
                  className="mb-2 px-3 text-xs font-semibold uppercase tracking-wider text-muted-foreground hover:text-foreground transition-colors flex items-center gap-1.5 w-full"
                  aria-expanded={!isCollapsed}
                  aria-controls={`category-${category.name}`}
                >
                  <ChevronRight
                    className={`h-3 w-3 transition-transform duration-200 ease-in-out ${
                      isCollapsed ? '' : 'rotate-90'
                    }`}
                    aria-hidden="true"
                  />
                  {category.displayName}
                </button>
              ) : null}

              {(!category.name || !isCollapsed) && (
                <ul className="space-y-1" id={category.name ? `category-${category.name}` : undefined}>
                  {category.items.map((item) => {
                    const href = `/docs/${item.slug}`;
                    const isActive = pathname === href;

                    return (
                      <li key={item.slug}>
                        <Link
                          href={href}
                          className={`block rounded-md px-3 py-2 text-sm transition-colors ${
                            isActive
                              ? "bg-accent text-accent-foreground font-medium"
                              : "text-muted-foreground hover:bg-accent/50 hover:text-foreground"
                          }`}
                        >
                          {item.title}
                        </Link>
                      </li>
                    );
                  })}
                </ul>
              )}
            </div>
          );
        })}
      </nav>
    </aside>
  );
}
```

### DocsContent

```tsx
// FRAMEWORK: This is the Next.js App Router version (server component).
// For Vite/TanStack: use a client component with @mdx-js/react or unified pipeline.
import { MDXRemote } from "next-mdx-remote/rsc";
import { CodeBlock } from "./CodeBlock";
import remarkGfm from "remark-gfm";
import rehypeSlug from "rehype-slug";

interface DocsContentProps {
  content: string;
}

export function DocsContent({ content }: DocsContentProps) {
  return (
    <div className="prose prose-lg prose-invert max-w-none">
      <MDXRemote
        source={content}
        options={{
          mdxOptions: {
            remarkPlugins: [remarkGfm],
            rehypePlugins: [rehypeSlug],
          },
        }}
        components={{
          code: CodeBlock,
        }}
      />
    </div>
  );
}
```

**Vite/TanStack alternative for DocsContent:**

```tsx
import { useEffect, useState } from "react";
import { unified } from "unified";
import remarkParse from "remark-parse";
import remarkGfm from "remark-gfm";
import remarkRehype from "remark-rehype";
import rehypeSlug from "rehype-slug";
import rehypeReact from "rehype-react";
import * as jsxRuntime from "react/jsx-runtime";
import { CodeBlock } from "./CodeBlock";

// Additional deps for Vite: npm install unified remark-parse remark-rehype rehype-react

interface DocsContentProps {
  content: string;
}

export function DocsContent({ content }: DocsContentProps) {
  const [rendered, setRendered] = useState<React.ReactNode>(null);

  useEffect(() => {
    unified()
      .use(remarkParse)
      .use(remarkGfm)
      .use(remarkRehype)
      .use(rehypeSlug)
      .use(rehypeReact, {
        jsx: jsxRuntime.jsx,
        jsxs: jsxRuntime.jsxs,
        Fragment: jsxRuntime.Fragment,
        components: { code: CodeBlock },
      } as Parameters<typeof rehypeReact>[0])
      .process(content)
      .then((file: { result: React.ReactNode }) => setRendered(file.result));
  }, [content]);

  return (
    <div className="prose prose-lg prose-invert max-w-none">
      {rendered}
    </div>
  );
}
```

---

## Step 5: Create Routes

### Next.js App Router

Create a route group `app/(docs)/` with these files:

**app/(docs)/layout.tsx**

```tsx
import { getDocNavCategories } from "@/lib/docs/loadDocs";
import { DocsSidebar } from "./_components/DocsSidebar";

export default async function DocsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const categories = await getDocNavCategories();

  return (
    <div className="min-h-screen">
      <DocsSidebar categories={categories} />
      <main className="ml-64 p-8">{children}</main>
    </div>
  );
}
```

**app/(docs)/docs/page.tsx**

```tsx
import { redirect } from "next/navigation";
import { getAllDocSlugs } from "@/lib/docs/loadDocs";

export default async function DocsIndexPage() {
  const slugs = await getAllDocSlugs();
  const firstDoc = slugs[0] || "getting-started";
  redirect(`/docs/${firstDoc}`);
}
```

**app/(docs)/docs/[...slug]/page.tsx**

```tsx
import { notFound } from "next/navigation";
import type { Metadata } from "next";
import { getAllDocSlugs, getDocBySlug } from "@/lib/docs/loadDocs";
import { DocsContent } from "../../_components/DocsContent";
import { TableOfContents } from "../../_components/TableOfContents";
import { extractHeadings } from "@/lib/docs/extractHeadings";

interface DocPageProps {
  params: Promise<{ slug: string[] }>;
}

export async function generateStaticParams() {
  const slugs = await getAllDocSlugs();
  return slugs.map((slug) => ({
    slug: slug.split("/"),
  }));
}

export async function generateMetadata({
  params,
}: DocPageProps): Promise<Metadata> {
  const { slug } = await params;
  const fullSlug = slug.join("/");
  const doc = await getDocBySlug(fullSlug);

  if (!doc) {
    return { title: "Not Found" };
  }

  return {
    title: doc.metadata.title,
    description: doc.metadata.description,
  };
}

export default async function DocPage({ params }: DocPageProps) {
  const { slug } = await params;
  const fullSlug = slug.join("/");
  const doc = await getDocBySlug(fullSlug);

  if (!doc) {
    notFound();
  }

  const headings = extractHeadings(doc.content);

  return (
    <div className="flex gap-8">
      <div className="min-w-0 max-w-4xl w-full">
        <DocsContent content={doc.content} />
      </div>
      {headings.length > 0 && <TableOfContents headings={headings} />}
    </div>
  );
}
```

### Vite + React Router

Create these route files in your `src/routes/` or `src/pages/` directory:

**src/routes/docs.tsx** (layout)

```tsx
import { Outlet, useLoaderData } from "react-router-dom";
import { DocsSidebar } from "../components/docs/DocsSidebar";

// Note: For Vite, doc loading must happen via API route or build-time.
// Option A: Create an API endpoint that calls getDocNavCategories()
// Option B: Use a Vite plugin to load at build time
// Option C: Fetch docs client-side from a /api/docs endpoint

export default function DocsLayout() {
  const { categories } = useLoaderData();

  return (
    <div className="min-h-screen">
      <DocsSidebar categories={categories} />
      <main className="ml-64 p-8">
        <Outlet />
      </main>
    </div>
  );
}
```

**src/routes/docs/$slug.tsx** (catch-all)

```tsx
import { useLoaderData } from "react-router-dom";
import { DocsContent } from "../components/docs/DocsContent";
import { TableOfContents } from "../components/docs/TableOfContents";

export default function DocPage() {
  const { doc, headings } = useLoaderData();

  return (
    <div className="flex gap-8">
      <div className="min-w-0 max-w-4xl w-full">
        <DocsContent content={doc.content} />
      </div>
      {headings.length > 0 && <TableOfContents headings={headings} />}
    </div>
  );
}
```

For Vite, you need a server endpoint or build-time data loading to read markdown files. Common approaches:
- **SSR mode** (Vite + Express): Create `/api/docs` and `/api/docs/:slug` endpoints using the loadDocs utilities
- **SPA mode**: Pre-build docs to JSON at build time with a Vite plugin
- **Full-stack** (TanStack Start): Use server functions directly

### TanStack Start

**src/routes/docs.tsx** (layout)

```tsx
import { createFileRoute, Outlet } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/start";
import { getDocNavCategories } from "../lib/docs/loadDocs";
import { DocsSidebar } from "../components/docs/DocsSidebar";

const fetchCategories = createServerFn("GET", async () => {
  return getDocNavCategories();
});

export const Route = createFileRoute("/docs")({
  loader: () => fetchCategories(),
  component: DocsLayout,
});

function DocsLayout() {
  const categories = Route.useLoaderData();

  return (
    <div className="min-h-screen">
      <DocsSidebar categories={categories} />
      <main className="ml-64 p-8">
        <Outlet />
      </main>
    </div>
  );
}
```

**src/routes/docs/$slug.tsx**

```tsx
import { createFileRoute } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/start";
import { getDocBySlug } from "../lib/docs/loadDocs";
import { extractHeadings } from "../lib/docs/extractHeadings";
import { DocsContent } from "../components/docs/DocsContent";
import { TableOfContents } from "../components/docs/TableOfContents";

const fetchDoc = createServerFn("GET", async (slug: string) => {
  const doc = await getDocBySlug(slug);
  if (!doc) throw new Error("Not found");
  const headings = extractHeadings(doc.content);
  return { doc, headings };
});

export const Route = createFileRoute("/docs/$slug")({
  loader: ({ params }) => fetchDoc(params.slug),
  component: DocPage,
});

function DocPage() {
  const { doc, headings } = Route.useLoaderData();

  return (
    <div className="flex gap-8">
      <div className="min-w-0 max-w-4xl w-full">
        <DocsContent content={doc.content} />
      </div>
      {headings.length > 0 && <TableOfContents headings={headings} />}
    </div>
  );
}
```

---

## Step 6: Styling Notes

- Requires `@tailwindcss/typography` plugin for `prose` classes
- Uses `prose prose-invert` for dark-mode markdown rendering. Adapt to project theme:
  - Light mode only: `prose` (no `prose-invert`)
  - Dark mode toggle: `prose dark:prose-invert`
- Headings need `scroll-margin-top` for TOC offset if you have a sticky header:
  ```css
  h2, h3, h4 { scroll-margin-top: 5rem; }
  ```
- Sidebar uses standard Tailwind color classes (`bg-background`, `text-foreground`, `text-muted-foreground`, `bg-accent`). If using shadcn/ui, these are already defined. If not, define them in your CSS or replace with direct Tailwind colors.
- The sidebar is `fixed left-0 top-0 w-64`. If you have a global nav/header, adjust `top-0` to match header height (e.g., `top-14` for a 3.5rem header).

---

## Step 7: Create Initial Content

### docs/01-progress.md

```markdown
# Project Progress

A living log of project development. Newest entries first.

---

### Docs Viewer Bootstrap

Bootstrapped the docs viewer with sidebar navigation, syntax highlighting, and table of contents. Created initial project documentation structure.

**Key files:**
- `lib/docs/` - Document loading utilities
- `docs/01-progress.md` - This progress log
- `docs/features/_TEMPLATE.md` - Feature documentation template

---
```

### docs/features/_TEMPLATE.md

```markdown
# Feature: [Feature Name]

## Overview
Brief description of what this feature does and its purpose

## Architecture
- High-level architecture approach
- Key libraries/services used
- Integration points

## Key Files
- `path/to/component.tsx` - Component description
- `path/to/hook.ts` - Hook description
- `path/to/action.ts` - Server action description

## Data Flow
1. User action
2. Processing step
3. State update
4. Result

## State Management
- How state is managed (context, zustand, server state, etc.)
- State access patterns

## API/Interface
### Components
- `ComponentName` - Description and key props

### Hooks
- `useHookName()` - Description and return type

### Actions/Utilities
- `functionName()` - Description and parameters

## Patterns & Conventions
- Coding patterns specific to this feature
- Error handling approach
- Testing patterns

## Related Features
- Feature dependencies
- Features that depend on this

## Testing
- Test file locations
- Key test scenarios
- How to run tests

## Future Enhancements
- Planned improvements
- Known limitations
```

---

## Step 8: Create Shipit Command

Write the following to `.claude/commands/shipit.md` in the project:

````markdown
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
````

---

## Step 9: Verification Checklist

After completing all steps, verify:

- [ ] `npm run build` (or equivalent) succeeds with no errors
- [ ] `/docs` route renders and redirects to first document
- [ ] Sidebar shows "Progress" as a navigable link
- [ ] Code blocks in progress page have syntax highlighting
- [ ] Table of contents appears on xl+ screens when H2 headings exist
- [ ] Collapsing/expanding sidebar categories works and persists across page loads
- [ ] `.claude/commands/shipit.md` exists in the project
- [ ] `docs/features/_TEMPLATE.md` exists but is NOT shown in sidebar navigation

If the build fails, fix any import path issues (the most common problem is `@/` alias not matching the project's tsconfig paths).
