# Adapter — vibe-stack-rules

> **Load this only when `stack/*.md` is present at repo root.** Otherwise the generator stays vanilla Next.js + Tailwind + shadcn.
>
> This file adds the layering that makes generated code drop straight into a vibe-stack-rules project. Every rule here is derived from `template/stack/*.md` — if those change, this file needs to change too.

---

## Activation contract

```
IF exists(stack/00-foundations.md)
   OR exists(stack/10-server.md)
   OR exists(stack/20-supabase.md)
   OR exists(stack/30-client.md)
   OR exists(stack/90-discipline.md)
THEN:
   adapter = "vibe-stack-rules"
   (load this file; apply all rules below in addition to the universal rules in synthesize-and-generate.md)
```

Announce once, up front: *"Detected vibe-stack-rules contract at `<path>/stack/`. Applying adapter layer."*

---

## Rule 1 — `page.tsx` is never `async`

**Source:** `stack/10-server.md` — static-shell contract.

```tsx
// ❌ What Claude Design's HTML might imply
export default async function LandingPage() {
  const data = await getHeroData();
  return <Hero data={data} />;
}

// ✅ What the adapter emits
import { Suspense } from "react";
import { Hero } from "./_components/hero";
import { HeroSkeleton } from "./_components/hero.skeleton";

export default function LandingPage() {
  return (
    <main className="mx-auto max-w-6xl px-6 py-12">
      <Suspense fallback={<HeroSkeleton />}>
        <Hero />
      </Suspense>
    </main>
  );
}
```

`page.tsx` fetches nothing, awaits nothing, is never `async`. It wires layout + Suspense boundaries. That's it.

---

## Rule 2 — Sibling skeletons are mandatory for every data-fetching component

**Source:** `stack/10-server.md`.

Naming is strict:

```
app/(marketing)/landing/_components/
├── hero.tsx                    ← async, fetches data
└── hero.skeleton.tsx           ← exports HeroSkeleton
```

Skeleton rules:

- Use shadcn `<Skeleton>` primitives for placeholder blocks.
- Match the real component's layout (same wrapper, same width/height classes) so the page doesn't jump when data arrives.
- No animations beyond the default shadcn shimmer.
- Never conditional — the skeleton file always exports its skeleton function unconditionally.

---

## Rule 3 — Push `"use client"` as deep as possible

**Source:** `stack/10-server.md` — RSC boundary.

If the design includes an interactive element (toggle, carousel, tab), extract **just that element** into a client component. Don't mark the whole section client.

Prefer the composition pattern:

```tsx
// Server component (default) — does layout + fetch
export async function FeatureSection() {
  const items = await getFeatures();
  return (
    <ClientCarousel>
      {items.map(i => <FeatureCard key={i.id} item={i} />)}
    </ClientCarousel>
  );
}

// ClientCarousel.tsx — small client island
"use client";
import { useState } from "react";
export function ClientCarousel({ children }: { children: React.ReactNode }) {
  const [index, setIndex] = useState(0);
  // ...
}
```

---

## Rule 4 — Mutations become Server Action stubs returning `ActionResult`

**Source:** `stack/10-server.md` + `stack/90-discipline.md` (single `ActionResult` shape).

If the design shows a form:

1. Emit the zod schema in `lib/validators/<domain>.ts`.
2. Emit the Server Action in `lib/db/mutations/<domain>.ts` with a `// TODO: wire DB` stub.
3. Client form uses `useActionState` + `zodResolver(<sameSchema>)` — one schema, two uses.
4. Do **not** emit a route handler in `app/api/*` for form submits. Route handlers are a last resort.

`ActionResult` is imported, not redeclared:

```ts
// types/index.ts (create if missing)
export type ActionResult<T = void> =
  | { ok: true; data?: T }
  | { ok: false; error?: string; fieldErrors?: Record<string, string[]> };
```

---

## Rule 5 — Route strings become `ROUTES` entries

**Source:** `stack/90-discipline.md`.

Before writing any `<Link href="...">` or `redirect("...")`, add or reuse an entry in `lib/constants.ts`:

```ts
// lib/constants.ts
export const ROUTES = {
  home: "/",
  landing: "/",              // example: landing page design
  // ... existing entries
} as const;
```

Then:

```tsx
import Link from "next/link";
import { ROUTES } from "@/lib/constants";
<Link href={ROUTES.landing}>Home</Link>
```

Never leave string literals for paths in JSX.

---

## Rule 6 — Zustand is client-only, UI-only

**Source:** `stack/30-client.md`.

If the design implies client state (a sidebar toggle, an open modal), use Zustand. But:

- Never use Zustand for data from the design (testimonials list, pricing tiers, etc.) — that's server data, passed as props.
- Always use the Store Factory + Provider pattern (see `stack/30-client.md`). No module-scope `create(...)`.
- Always call hooks with a selector: `useUiStore(s => s.field)`.

If the design doesn't imply any cross-component client state, **do not introduce Zustand**. YAGNI.

---

## Rule 7 — Auth gates go in the `(app)` layout, not in individual pages

**Source:** `stack/20-supabase.md`.

If the design is marked as an authenticated route (dashboard, settings, etc.):

- Place the file under `app/(app)/<route>/page.tsx`.
- Do NOT add auth checks inside the page — `app/(app)/layout.tsx` already handles them.
- Set `export const dynamic = "force-dynamic";` on the page (required for any route reading session cookies).
- If `app/(app)/layout.tsx` doesn't exist yet, emit it with the canonical auth gate from `stack/20-supabase.md`.

If the design is a public page (marketing/landing), place it under `app/(marketing)/` or the root `app/` segment. Public pages do NOT get `force-dynamic` — let Next's default caching apply.

---

## Rule 8 — Single source of truth — check before declaring

**Source:** `stack/90-discipline.md`.

Before the Generator writes any of these, it **must grep** the existing repo for:

| Kind | Where it should live | Grep before declaring |
|---|---|---|
| Zod schema | `lib/validators/*.ts` | Grep `z.object.*<similar field set>` |
| Type | `types/database.ts` (if DB-derived), `types/index.ts` (otherwise) | Grep `type <Name>` |
| Route | `lib/constants.ts` → `ROUTES` | Grep `"/<path>"` |
| Constant | `lib/constants.ts` | Grep the literal value |
| cn helper | `lib/utils.ts` | Confirm it exports `cn` |
| ActionResult | `types/index.ts` | Grep `type ActionResult` |

If it already exists, import it. Never declare a parallel version.

---

## Rule 9 — Query functions live in `lib/db/queries/*`, not in components

**Source:** `stack/20-supabase.md`.

If the generated component is going to fetch data, emit a query-function stub:

```ts
// lib/db/queries/<domain>.ts
import "server-only";
import type { SupabaseClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database";

type Client = SupabaseClient<Database>;

export async function get<Name>(supabase: Client) {
  // TODO: wire real Supabase query
  // const { data, error } = await supabase.from("<table>").select("...");
  // if (error) throw error;
  // return data;
  return [] as const; // placeholder
}
```

The component then calls it:

```tsx
export async function Hero() {
  const supabase = await createClient();
  const data = await getHero(supabase);
  return <section>...</section>;
}
```

Per-request Supabase client, created **inside** the component function. Never at module scope.

---

## Rule 10 — Domain wrappers > raw shadcn trees

**Source:** `stack/30-client.md`.

If the Component Extractor (Phase 2) identified a 3+ repeating `<Card>...<CardHeader>...` tree, the adapter enforces that the wrapper lands in `components/common/*`, not in a route's `_components/`. Route-private wrappers are only for patterns that stay in that route.

---

## Generation checklist for adapter mode

Before Phase 4 writes a single file, the Generator confirms:

- [ ] The chosen `app/` segment matches the design's auth intent (marketing group vs. app group).
- [ ] Every data-fetching component has a planned skeleton sibling.
- [ ] `page.tsx` imports match available components (including any new ones being created in this run).
- [ ] `lib/constants.ts` has entries for every route string used.
- [ ] `lib/validators/` and `lib/db/queries/` have the stubs needed for any forms or lists.
- [ ] Zod schemas are shared between validators and forms.
- [ ] No module-scope Supabase client anywhere in the diff.
- [ ] `"use client"` appears only in leaf interactive components.
- [ ] shadcn install command includes `skeleton` (because we just required skeletons).

If any box is unchecked, fix the plan before writing.

---

## Anti-patterns specific to this adapter

| Anti-pattern (what the HTML might imply) | What the adapter emits instead |
|---|---|
| `<form onsubmit="submitForm()">` | Server Action + `useActionState` + zod schema in `lib/validators/` |
| `<script>fetch('/api/...')</script>` in the HTML | Server Component with per-request Supabase + query in `lib/db/queries/` |
| Route handler (`app/api/...route.ts`) for a simple mutation | Server Action |
| `useEffect(() => fetch(...))` | Move to server, or emit a Server Action call on interaction |
| Hard-coded pricing/feature data in the component | `PRICING` / `FEATURES` constants in `lib/constants.ts`, imported by the component |
| Auth check inside a page component | Remove — the `(app)/layout.tsx` handles it |
| Session read via `getSession()` | Always `getUser()` on the server |
| `revalidate = N` on a route that reads session | Delete it; add `dynamic = "force-dynamic"` instead |

---

## When this adapter should NOT be applied

- The user explicitly opts out: *"skip the vibe-stack-rules layer"* in the original prompt.
- The design is a one-off experiment in a sandbox repo that happens to have `stack/` but where the user is iterating on pure aesthetics.

In those cases, announce the opt-out, run in vanilla mode, and note it in the verify report so the user has a trail.
