# Stack Rules — Foundations

> **Audience:** Claude Code / any LLM writing or modifying code in this repo.
>
> **Posture:** This file and its siblings in `stack/` are **law**, not advice. If a rule here conflicts with a pattern you remember from training data, **this file wins**. If a rule here conflicts with a request from the user, **ask before breaking it**.

---

## How to read these files

1. Before writing any code, re-read the relevant `stack/*.md` file for the area you're touching.
2. **Never invent a file.** If you need a client, schema, type, or constant, search the repo first. Duplicates are the #1 failure mode in LLM-generated codebases.
3. **Never fetch in `page.tsx` without `<Suspense>` around the fetch.** Static shell renders first. Always. See `10-server.md`.
4. **Never put a Supabase client in module scope.** Always inside a function. See `20-supabase.md`.
5. **Never use Zustand for server data** (user lists, DB rows, API results). That's React Server Components + Suspense territory. Zustand is for client UI state only. See `30-client.md`.
6. **Never re-declare a type, constant, schema, or string literal that exists elsewhere in the repo.** Import it. See `90-discipline.md`.
7. When in doubt, prefer **less code, more server, more composition**.

---

## Version pins (assume these unless `package.json` says otherwise)

- `next` >= 15 (App Router only — Pages Router is dead to us)
- `react` >= 19
- `typescript` — strict mode on, `noUncheckedIndexedAccess` on
- `@supabase/ssr` >= 0.10 (the `@supabase/auth-helpers-*` packages are **deprecated** — never install them)
- `@supabase/supabase-js` >= 2.45
- `zustand` >= 5
- `tailwindcss` >= 3.4 (v4 if the project is on it)
- `shadcn` CLI: `npx shadcn@latest add <component>` — never hand-write a shadcn component from memory

**Middleware file naming:** Next.js ≤15 uses `middleware.ts` at root. Next.js 16+ renamed it to `proxy.ts`. Check `package.json` before creating the file. The code inside is identical — only the filename changes.

---

## Core principles (the non-negotiables)

| # | Principle | Why |
|---|-----------|-----|
| 1 | **Server Components by default.** `"use client"` is an escape hatch, not a starting point. | Smaller bundles, direct DB access, no waterfalls. |
| 2 | **No API calls inside `page.tsx` on the critical render path.** Static shell streams first; data streams in via `<Suspense>` + skeletons. | TTFB stays fast. Page is never a white screen. |
| 3 | **Every API-bound component (cards, lists, tables, charts) ships with a sibling `<ComponentName>Skeleton />`** in the same folder. | Skeleton renders as Suspense fallback. No conditional spinners scattered through JSX. |
| 4 | **Single source of truth for every type, schema, constant, and query.** If it's declared twice, one of them is wrong. | Prevents the #1 LLM bug: silent drift between two definitions of the "same" thing. |
| 5 | **Zustand is client-only and UI-only.** Never a cache for server data. | Prevents cross-request state leaks on the server and stale data on the client. |
| 6 | **Supabase clients are created per-request, never at module scope.** | Vercel Fluid Compute and long-lived server processes will leak one user's session into another user's request otherwise. |
| 7 | **Mutations go through Server Actions** (`"use server"`) unless you have a concrete reason to expose a Route Handler. | One code path, type-safe, progressive-enhancement friendly. |
| 8 | **Styling is 100% shadcn primitives + Tailwind utilities + `cn()`.** No raw CSS files, no CSS modules, no styled-components, no inline `style=` objects for anything themeable. | Consistency, theming, dark mode, accessibility come for free. |

---

## Canonical project structure

```
src/
  app/                         # Next.js App Router
    (marketing)/               # Route group — public pages
    (app)/                     # Route group — authenticated shell
      layout.tsx               # Auth gate lives here (see 20-supabase.md)
      dashboard/
        page.tsx               # Static shell + Suspense boundaries only
        loading.tsx            # Route-level skeleton (optional — prefer per-component)
        error.tsx              # Route-level error boundary
        _components/           # Route-private components (underscore = not a route)
          revenue-card.tsx
          revenue-card.skeleton.tsx
    api/
      */route.ts               # Only when a Server Action cannot do the job
    layout.tsx                 # Root layout — providers mounted here
    globals.css                # Tailwind + shadcn CSS vars only

  components/
    ui/                        # shadcn primitives — DO NOT hand-edit, use `shadcn add`
    common/                    # Shared domain components (Navbar, PageHeader, EmptyState)
    forms/                     # Form components built on shadcn Form + react-hook-form + zod

  lib/
    supabase/
      client.ts                # createBrowserClient  (client components)
      server.ts                # createServerClient   (RSC, actions, route handlers)
      middleware.ts            # updateSession helper (used by proxy.ts/middleware.ts)
      admin.ts                 # service-role client — SERVER ONLY, never imported by client code
    db/
      queries/                 # Server-only data-access functions (one file per domain)
      mutations/               # Server Actions grouped by domain
    validators/                # Zod schemas — the SINGLE source of truth for shapes
    utils.ts                   # cn(), formatters, pure helpers
    constants.ts               # App-wide constants (routes, enums, limits)

  stores/                      # Zustand stores (client-only state)
    ui-store.ts
    <feature>-store.ts

  types/
    database.ts                # Generated from Supabase: `supabase gen types typescript`
    index.ts                   # Re-exports + app-level types derived from database.ts

  hooks/                       # Client-only React hooks
middleware.ts | proxy.ts       # Root — calls lib/supabase/middleware.ts
```

**Rules for this tree:**

- `_components/` folders (underscore prefix) are private to a route and **not allowed to be imported from another route**. If two routes need the same component, it lives in `src/components/common/`.
- `lib/db/queries/*` are `async` functions that take a Supabase client as their first arg and return plain data. They are the **only** code that writes Supabase queries. Components never call `supabase.from(...)` directly.
- `types/database.ts` is generated. **Never hand-edit it.** Regenerate with `pnpm db:types` (or whatever the project's script is).

---

## When these files are wrong

If a rule in any `stack/*.md` file is actively blocking you from shipping the right thing:

1. **Stop.** Don't silently route around it.
2. **Tell the user** which rule is in the way and why.
3. **Propose an amendment** to the relevant stack file before writing the code that breaks the rule.
4. Only proceed once the user has explicitly agreed.

These files are the contract. The contract can be changed — but not quietly.
