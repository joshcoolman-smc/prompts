# Minimal Supabase Authentication with TanStack Start - Implementation Plan

## Overview

This plan provides a comprehensive guide for implementing a minimal, production-ready authentication system using Supabase and TanStack Start. Based on the GenZen codebase architecture, this implementation includes:

- Email/password login route at `/login`
- Protected dashboard route at `/dashboard` with user information display
- Sidebar navigation (desktop) and mobile sheet navigation
- Session persistence and auth state management
- Clean, minimal structure that integrates into your existing app

**Important Notes:**
- **Non-invasive:** This plan adds authentication routes (`/login`, `/dashboard`) without modifying your existing pages. Your home page and other routes remain untouched.
- **Integration:** How you link to the login page is up to you - add it to your nav, create a "Sign In" button, or keep it accessible only via direct URL.
- **Tech Stack:** Built for TanStack Start with Tailwind CSS. Component examples use Tailwind utility classes.

## Architecture Summary

### Authentication Flow

```
App Load → AuthProvider initializes → Get session from Supabase
         → Subscribe to auth changes → Route protection via useEffect
         → Login → Dashboard with user info
```

### Component Hierarchy

```
RootDocument (HTML shell)
└── RootComponent
    └── AuthProvider (manages auth state globally)
        └── <Outlet /> (routes)
            ├── Your existing routes (/, /about, etc.) - Unchanged
            ├── LoginPage (/login) - Public authentication route
            └── DashboardLayoutRoute (/dashboard) - Protected
                └── DashboardLayout (sidebar + content wrapper)
                    └── <Outlet /> (nested routes)
                        ├── /dashboard/index - Home with user info
                        └── /dashboard/profile - Profile page
```

### Key Patterns

1. **Manual Route Protection**: Uses `useEffect` with `useAuth()` hook to redirect unauthenticated users
2. **Context-Based Auth**: React Context wraps entire app, provides auth state everywhere
3. **Session Persistence**: Supabase automatically stores JWT in localStorage
4. **Real-time Updates**: `onAuthStateChange` subscription keeps UI in sync with auth state

---

## Critical Files & Implementation Order

### Phase 1: Core Setup (Files 1-4)

**1. `/src/lib/supabase.ts` - Supabase Client**

Creates singleton Supabase client with environment variables.

```typescript
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

if (!supabaseUrl || !supabaseAnonKey) {
  throw new Error("Missing Supabase environment variables");
}

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

**2. `/src/lib/auth.ts` - Auth Context & Hook**

Defines TypeScript types and hook for accessing auth throughout app.

```typescript
import { createContext, useContext } from "react";
import type { User, Session } from "@supabase/supabase-js";

export interface AuthContextType {
  user: User | null;
  session: Session | null;
  loading: boolean;
  signIn: (email: string, password: string) => Promise<{ error: Error | null }>;
  signOut: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextType | null>(null);

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

**3. `/src/lib/utils.ts` - Utility Functions**

Merges Tailwind classes conditionally.

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**4. `/.env.local` - Environment Variables**

```bash
VITE_SUPABASE_URL=your_supabase_project_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
```

Add to `.gitignore`:
```
.env.local
```

### Phase 2: Authentication Provider (File 5)

**5. `/src/components/auth-provider.tsx` - Auth State Management**

Most critical file - manages authentication state for entire app.

Key implementation details:
- Three state variables: `user`, `session`, `loading` (starts as `true`)
- `useEffect` on mount: Gets initial session + subscribes to auth changes
- Cleanup: Unsubscribes on unmount to prevent memory leaks

```typescript
import { useState, useEffect, type ReactNode } from "react";
import { AuthContext } from "@/lib/auth";
import { supabase } from "@/lib/supabase";
import type { User, Session } from "@supabase/supabase-js";

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [session, setSession] = useState<Session | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Get initial session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      setUser(session?.user ?? null);
      setLoading(false);
    });

    // Subscribe to auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (_event, session) => {
        setSession(session);
        setUser(session?.user ?? null);
        setLoading(false);
      }
    );

    return () => subscription.unsubscribe();
  }, []);

  const signIn = async (email: string, password: string) => {
    const { error } = await supabase.auth.signInWithPassword({
      email,
      password,
    });
    return { error };
  };

  const signOut = async () => {
    await supabase.auth.signOut();
  };

  return (
    <AuthContext.Provider
      value={{ user, session, loading, signIn, signOut }}
    >
      {children}
    </AuthContext.Provider>
  );
}
```

### Phase 3: Styling (File 6)

**6. `/src/styles.css` - Tailwind Setup**

```css
@import "tailwindcss";

/* Add your custom styles and design tokens here */
```

### Phase 4: Root Route (File 7)

**7. `/src/routes/__root.tsx` - App Wrapper with AuthProvider**

Critical: This wraps the entire app with AuthProvider, making auth state available everywhere.

```typescript
import { createRootRoute, Outlet } from "@tanstack/react-router";
import { AuthProvider } from "@/components/auth-provider";
import appCss from "../styles.css?url";

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: "utf-8" },
      { name: "viewport", content: "width=device-width, initial-scale=1" },
      { title: "My App" },
    ],
    links: [{ rel: "stylesheet", href: appCss }],
  }),
  component: RootComponent,
});

function RootComponent() {
  return (
    <AuthProvider>
      <Outlet />
    </AuthProvider>
  );
}
```

### Phase 5: Login Page (File 8)

**8. `/src/routes/login.tsx` - Login Route**

A dedicated login route at `/login`. This does not replace your home page - it's a separate route that you can link to however you want in your app.

**Note:** This implementation only includes sign-in functionality. Users are expected to be created through the Supabase admin dashboard, an invite system, or other backend processes. If you need public sign-up, see the "Extension Points" section for how to add it.

Key patterns:
- Check `loading` state first, show loading UI
- Redirect if already authenticated: `if (user) navigate("/dashboard")`
- Form submission calls `signIn()` from `useAuth()`
- Display errors returned from Supabase
- Disable submit button while `submitting`

```typescript
import { createFileRoute, useNavigate } from "@tanstack/react-router";
import { useState } from "react";
import { useAuth } from "@/lib/auth";

export const Route = createFileRoute("/login")({
  component: LoginPage,
});

function LoginPage() {
  const { user, loading, signIn } = useAuth();
  const navigate = useNavigate();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);

  // Show loading during initial auth check
  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-muted-foreground">Loading...</div>
      </div>
    );
  }

  // Redirect if already authenticated
  if (user) {
    navigate({ to: "/dashboard" });
    return null;
  }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setSubmitting(true);

    const { error } = await signIn(email, password);
    setSubmitting(false);

    if (error) {
      setError(error.message);
    } else {
      navigate({ to: "/dashboard" });
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-background p-4">
      <div className="w-full max-w-md space-y-8">
        <div className="text-center">
          <h1 className="text-3xl font-bold">Welcome Back</h1>
          <p className="text-muted-foreground mt-2">
            Sign in to your account
          </p>
        </div>

        <form onSubmit={handleSubmit} className="space-y-6">
          <div className="space-y-2">
            <label htmlFor="email" className="text-sm font-medium">
              Email
            </label>
            <input
              id="email"
              type="email"
              required
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full px-3 py-2 bg-card border border-input rounded-md"
            />
          </div>

          <div className="space-y-2">
            <label htmlFor="password" className="text-sm font-medium">
              Password
            </label>
            <input
              id="password"
              type="password"
              required
              minLength={6}
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full px-3 py-2 bg-card border border-input rounded-md"
            />
          </div>

          {error && (
            <div className="text-sm text-red-500 bg-red-500/10 px-3 py-2 rounded-md">
              {error}
            </div>
          )}

          <button
            type="submit"
            disabled={submitting}
            className="w-full py-2 bg-primary hover:bg-primary/90 text-primary-foreground font-medium rounded-md disabled:opacity-50"
          >
            {submitting ? "Signing in..." : "Sign in"}
          </button>
        </form>
      </div>
    </div>
  );
}
```

### Phase 6: Layout Components (Files 9-11)

**9. `/src/components/Sidebar.tsx` - Desktop Navigation**

Fixed left sidebar with navigation links.

Key patterns:
- Fixed positioning: `fixed left-0 top-0 h-screen w-64`
- Hidden on mobile: `hidden lg:flex`
- Active link detection using `useLocation()` from TanStack Router
- Active styling: Left border + background highlight

```typescript
import { Link, useLocation } from "@tanstack/react-router";
import { Home, User } from "lucide-react";
import { cn } from "@/lib/utils";

const navItems = [
  { label: "Home", href: "/dashboard", icon: Home },
  { label: "Profile", href: "/dashboard/profile", icon: User },
];

export function Sidebar({ className = "" }: { className?: string }) {
  const location = useLocation();

  const isActive = (href: string) => {
    if (href === "/dashboard") {
      return location.pathname === "/dashboard";
    }
    return location.pathname.startsWith(href);
  };

  return (
    <aside
      className={cn(
        "fixed left-0 top-0 h-screen w-64 flex-col border-r border-border bg-card",
        className
      )}
    >
      {/* Logo/Header - Replace with your own branding */}
      <div className="flex h-16 items-center px-6 border-b border-border">
        <span className="text-lg font-semibold">
          App Name
        </span>
      </div>

      {/* Navigation */}
      <nav className="flex-1 space-y-2 p-4">
        {navItems.map((item) => {
          const active = isActive(item.href);
          const Icon = item.icon;

          return (
            <Link
              key={item.href}
              to={item.href}
              className={cn(
                "flex items-center gap-3 rounded-md px-3 py-2 text-sm transition-colors",
                active
                  ? "border-l-2 border-primary bg-muted"  // Use your own active styles
                  : "text-muted-foreground hover:bg-muted hover:text-foreground"
              )}
            >
              <Icon className="h-5 w-5" />
              <span>{item.label}</span>
            </Link>
          );
        })}
      </nav>
    </aside>
  );
}
```

**10. `/src/components/MobileNav.tsx` - Mobile Sheet Navigation**

Mobile drawer/sheet with same navigation items as Sidebar.

Requires `@radix-ui/react-dialog` for Sheet component. Can use a simple slide-in `<div>` if not using Radix UI.

Key pattern: Auto-close sheet on route change via `useEffect`.

```typescript
import { useState, useEffect } from "react";
import { Link, useLocation } from "@tanstack/react-router";
import { Menu, Home, User, X } from "lucide-react";
import { cn } from "@/lib/utils";

const navItems = [
  { label: "Home", href: "/dashboard", icon: Home },
  { label: "Profile", href: "/dashboard/profile", icon: User },
];

export function MobileNav({ className = "" }: { className?: string }) {
  const [open, setOpen] = useState(false);
  const location = useLocation();

  // Close on route change
  useEffect(() => {
    setOpen(false);
  }, [location.pathname]);

  return (
    <>
      {/* Menu button */}
      <button
        onClick={() => setOpen(true)}
        className={cn(
          "fixed left-4 top-4 z-50 p-2 bg-card border border-border rounded-md",
          className
        )}
      >
        <Menu className="h-6 w-6" />
      </button>

      {/* Sheet/Drawer */}
      {open && (
        <>
          {/* Backdrop */}
          <div
            className="fixed inset-0 bg-black/50 z-50"
            onClick={() => setOpen(false)}
          />

          {/* Sheet Content */}
          <div className="fixed left-0 top-0 h-screen w-64 bg-card border-r border-border z-50 flex flex-col">
            {/* Header - Replace with your own branding */}
            <div className="flex items-center justify-between px-6 h-16 border-b border-border">
              <span className="text-lg font-semibold">
                App Name
              </span>
              <button onClick={() => setOpen(false)}>
                <X className="h-5 w-5" />
              </button>
            </div>

            {/* Navigation */}
            <nav className="flex-1 space-y-2 p-4">
              {navItems.map((item) => {
                const Icon = item.icon;
                return (
                  <Link
                    key={item.href}
                    to={item.href}
                    className="flex items-center gap-3 rounded-md px-3 py-2 text-sm text-muted-foreground hover:bg-muted hover:text-foreground"
                  >
                    <Icon className="h-5 w-5" />
                    <span>{item.label}</span>
                  </Link>
                );
              })}
            </nav>
          </div>
        </>
      )}
    </>
  );
}
```

**11. `/src/components/DashboardLayout.tsx` - Layout Wrapper**

Combines Sidebar (desktop) + MobileNav (mobile) + main content area.

```typescript
import { Sidebar } from "./Sidebar";
import { MobileNav } from "./MobileNav";

export function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex min-h-screen bg-background">
      {/* Desktop sidebar - hidden on mobile */}
      <Sidebar className="hidden lg:flex" />

      {/* Mobile nav trigger - hidden on desktop */}
      <MobileNav className="lg:hidden" />

      {/* Main content with offset for desktop sidebar */}
      <main className="flex-1 p-6 lg:ml-64">{children}</main>
    </div>
  );
}
```

### Phase 7: Protected Dashboard Routes (Files 12-14)

**12. `/src/routes/dashboard.tsx` - Dashboard Layout Route with Auth Guard**

Critical pattern: Manual route protection using `useEffect` to redirect unauthenticated users.

```typescript
import { createFileRoute, useNavigate, Outlet } from "@tanstack/react-router";
import { useEffect } from "react";
import { useAuth } from "@/lib/auth";
import { DashboardLayout } from "@/components/DashboardLayout";

export const Route = createFileRoute("/dashboard")({
  component: DashboardLayoutRoute,
});

function DashboardLayoutRoute() {
  const { user, loading } = useAuth();
  const navigate = useNavigate();

  // Redirect to login if not authenticated
  useEffect(() => {
    if (!loading && !user) {
      navigate({ to: "/login" });
    }
  }, [user, loading, navigate]);

  // Show loading during auth check
  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-muted-foreground">Loading...</div>
      </div>
    );
  }

  // Don't render if no user (redirect happening)
  if (!user) {
    return null;
  }

  // Render layout with nested routes
  return (
    <DashboardLayout>
      <Outlet />
    </DashboardLayout>
  );
}
```

**Why this pattern:**
- TanStack Router doesn't have built-in route middleware/guards
- Uses React `useEffect` for client-side protection
- All child routes (`/dashboard/*`) automatically inherit protection
- Loading state prevents flash of wrong content
- Redirects to `/login` - adjust this path if your login route is different

**13. `/src/routes/dashboard/index.tsx` - Dashboard Home**

Displays user information from auth context + sign out button.

```typescript
import { createFileRoute } from "@tanstack/react-router";
import { useAuth } from "@/lib/auth";

export const Route = createFileRoute("/dashboard/")({
  component: DashboardHome,
});

function DashboardHome() {
  const { user, signOut } = useAuth();

  const handleSignOut = async () => {
    await signOut();
  };

  return (
    <div className="space-y-8">
      {/* Header with sign out button */}
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold">Dashboard</h1>
        <button
          onClick={handleSignOut}
          className="text-sm text-muted-foreground hover:text-foreground transition-colors"
        >
          Sign out
        </button>
      </div>

      {/* User info card */}
      <div className="bg-card rounded-lg border border-border p-6 space-y-4">
        <h2 className="font-medium text-lg">User Information</h2>
        <div className="space-y-3 text-sm">
          <div className="flex justify-between py-2 border-b border-border">
            <span className="text-muted-foreground">Email</span>
            <span className="text-foreground">{user?.email}</span>
          </div>
          <div className="flex justify-between py-2 border-b border-border">
            <span className="text-muted-foreground">User ID</span>
            <span className="text-foreground font-mono text-xs">
              {user?.id}
            </span>
          </div>
          <div className="flex justify-between py-2">
            <span className="text-muted-foreground">Created</span>
            <span className="text-foreground">
              {user?.created_at
                ? new Date(user.created_at).toLocaleDateString()
                : "—"}
            </span>
          </div>
        </div>
      </div>
    </div>
  );
}
```

**14. `/src/routes/dashboard/profile.tsx` - Profile Page**

Similar to dashboard home but focused on profile information.

```typescript
import { createFileRoute } from "@tanstack/react-router";
import { useAuth } from "@/lib/auth";

export const Route = createFileRoute("/dashboard/profile")({
  component: ProfilePage,
});

function ProfilePage() {
  const { user } = useAuth();

  return (
    <div className="space-y-8">
      <h1 className="text-2xl font-semibold">Profile</h1>

      <div className="bg-card rounded-lg border border-border p-6 space-y-4">
        <h2 className="font-medium text-lg">User Information</h2>
        <div className="space-y-3 text-sm">
          <div className="flex justify-between py-2 border-b border-border">
            <span className="text-muted-foreground">Email</span>
            <span className="text-foreground">{user?.email}</span>
          </div>
          <div className="flex justify-between py-2 border-b border-border">
            <span className="text-muted-foreground">User ID</span>
            <span className="text-foreground font-mono text-xs">
              {user?.id}
            </span>
          </div>
          <div className="flex justify-between py-2">
            <span className="text-muted-foreground">Created</span>
            <span className="text-foreground">
              {user?.created_at
                ? new Date(user.created_at).toLocaleDateString()
                : "—"}
            </span>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## Configuration Files

### `package.json` - Dependencies

```json
{
  "name": "your-auth-app",
  "type": "module",
  "scripts": {
    "dev": "vite dev --port 3000",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@supabase/supabase-js": "^2.94.1",
    "@tanstack/react-router": "^1.132.0",
    "@tanstack/react-start": "^1.132.0",
    "react": "^19.2.0",
    "react-dom": "^19.2.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^3.4.0",
    "lucide-react": "^0.563.0",
    "@radix-ui/react-dialog": "^1.1.15"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^5.0.4",
    "@tailwindcss/vite": "^4.0.6",
    "tailwindcss": "^4.0.6",
    "vite": "^7.1.7",
    "vite-tsconfig-paths": "^5.1.4",
    "typescript": "^5.7.2"
  }
}
```

### `vite.config.ts` - Vite Configuration

```typescript
import { defineConfig } from "vite";
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import viteReact from "@vitejs/plugin-react";
import viteTsConfigPaths from "vite-tsconfig-paths";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [
    viteTsConfigPaths({ projects: ["./tsconfig.json"] }),
    tailwindcss(),
    tanstackStart(),
    viteReact(),
  ],
});
```

### `tsconfig.json` - TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

---

## Supabase Configuration

### Required Supabase Setup

1. **Create Supabase Project:**
   - Go to https://supabase.com/dashboard
   - Click "New Project"
   - Note: Project URL, Anon Key (from Settings > API)

2. **Enable Email Authentication:**
   - Navigate to Authentication > Providers
   - Ensure "Email" provider is enabled
   - For testing: Disable "Enable email confirmations" in settings

3. **Create Test User (via Admin Dashboard):**
   - Go to Authentication > Users
   - Click "Add user" > "Create new user"
   - Enter: `test@example.com` / `password123`
   - **Note:** This is how you'll create users for your app. There's no public sign-up page in this implementation - users are managed through the admin dashboard or your own backend processes.

4. **Environment Variables:**
   ```bash
   VITE_SUPABASE_URL=https://xxxxx.supabase.co
   VITE_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   ```

---

---

## Verification & Testing

### End-to-End Verification Steps

1. **Start Development Server:**
   ```bash
   npm install
   npm run dev
   ```
   Open http://localhost:3000

2. **Test Authentication Flow:**
   - [ ] Visit `/login` - should see login page
   - [ ] Already logged in? Should redirect to `/dashboard`
   - [ ] Login with test credentials
   - [ ] Should redirect to `/dashboard` showing user info
   - [ ] Sign out button should log out (redirect behavior is up to you)

3. **Test Protected Routes:**
   - [ ] Visit `/dashboard` while logged out - should redirect to `/login`
   - [ ] Login, then visit `/dashboard` - should show dashboard
   - [ ] Refresh page - should stay logged in
   - [ ] Visit `/dashboard/profile` - should show profile

4. **Test Navigation:**
   - [ ] Desktop (>1024px): Sidebar should be visible
   - [ ] Mobile (<1024px): Hamburger menu should show
   - [ ] Click nav links - should navigate correctly
   - [ ] Active link should be highlighted
   - [ ] Mobile nav should close after clicking link

5. **Test Error Handling:**
   - [ ] Login with invalid credentials - should show error
   - [ ] Login with empty fields - should show validation
   - [ ] Network offline - should handle gracefully

6. **Test Session Persistence:**
   - [ ] Login and close browser
   - [ ] Reopen browser to `/dashboard`
   - [ ] Should still be logged in (localStorage)

### Browser DevTools Checks

**Network Tab:**
- Initial session check: `POST /auth/v1/token?grant_type=refresh_token`
- Login: `POST /auth/v1/token?grant_type=password`
- Logout: `POST /auth/v1/logout`

**Application Tab:**
- Check localStorage for `sb-xxxxx-auth-token`
- Token should exist when logged in
- Token should be cleared after logout

**Console:**
- No errors on page load
- No missing dependency warnings

### TypeScript Check

```bash
npx tsc --noEmit
```

Should report 0 errors.

---

## Common Issues & Solutions

### Issue: "Missing Supabase environment variables"

**Solution:**
- Ensure file is named `.env.local` (not `.env`)
- Restart dev server after creating/editing `.env.local`
- Variables must start with `VITE_` prefix

### Issue: Infinite redirect loop

**Cause:** Missing dependencies in `useEffect`

**Solution:**
```typescript
useEffect(() => {
  if (!loading && !user) {
    navigate({ to: "/" });
  }
}, [user, loading, navigate]); // All three required
```

### Issue: Auth state not persisting

**Solution:**
- Check localStorage is enabled
- Verify Supabase client is singleton (not recreated)
- Check browser console for CORS errors

### Issue: Tailwind classes not applying

**Solution:**
- Ensure `@tailwindcss/vite` plugin in `vite.config.ts`
- Import `styles.css` in `__root.tsx`
- Verify CSS custom properties in `:root`

---

## Extension Points

Once basic auth is working, these features can be added:

### Optional: Public Sign-Up

If you want users to be able to create their own accounts (rather than creating them through the admin dashboard), add a sign-up page:

1. **Add `signUp` to AuthContext** (`/src/lib/auth.ts`):
   ```typescript
   signUp: (email: string, password: string) => Promise<{ error: Error | null }>;
   ```

2. **Implement in AuthProvider** (`/src/components/auth-provider.tsx`):
   ```typescript
   const signUp = async (email: string, password: string) => {
     const { error } = await supabase.auth.signUp({ email, password });
     return { error };
   };
   // Add signUp to the context value
   ```

3. **Create Sign-Up Route** (`/src/routes/signup.tsx`):
   - Similar structure to login page at `/login`
   - Use `signUp()` instead of `signIn()`
   - Consider email confirmation flow in Supabase settings
   - This creates a route at `/signup` - how you link to it is up to you

### Other Extensions

1. **Password Reset** - Use `supabase.auth.resetPasswordForEmail()`
2. **User Profile Editing** - Update `user_metadata` via `supabase.auth.updateUser()`
3. **Database Tables** - Add custom tables with Row-Level Security policies
4. **Role-Based Access** - Check `user.user_metadata.role` for permissions
5. **OAuth Providers** - Add Google/GitHub login via Supabase providers
6. **Invite System** - Create backend endpoints to generate invite codes and create users

---

## Summary

This plan provides everything needed to implement a minimal, production-ready Supabase authentication system with:

- **Login page** with email/password authentication
- **Protected dashboard** with automatic route guards
- **User information display** from Supabase auth
- **Responsive navigation** with sidebar (desktop) and mobile drawer
- **Session persistence** via Supabase localStorage
- **Clean architecture** suitable for custom design systems

**Key Architecture Decisions:**
- Manual route protection via `useEffect` (no middleware)
- React Context for auth state management
- Client-side only (Supabase handles server validation via JWT)
- File-based routing with TanStack Router
- Tailwind CSS for styling

**Critical Files:** The 5 most important files to get right:
1. `/src/components/auth-provider.tsx` - Auth state management
2. `/src/routes/__root.tsx` - Wraps app with AuthProvider
3. `/src/routes/dashboard.tsx` - Route protection pattern
4. `/src/lib/supabase.ts` - Client configuration
5. `/src/routes/login.tsx` - Login flow

All patterns are based on the proven GenZen codebase architecture and can be reliably implemented by another developer or AI agent.
