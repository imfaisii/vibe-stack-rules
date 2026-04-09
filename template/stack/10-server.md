# Stack Rules — Server-Side: Rendering, Data, and Mutations

> Covers: the RSC boundary, the static-shell data-fetching contract, skeletons, caching, and Server Actions.

---

## The RSC boundary rule (memorize this)

```
┌─────────────────────────────────────────────────────────┐
│  Server Component (default)                             │
│    • async, awaits data directly                        │
│    • imports server-only code (DB, secrets, fs)         │
│    • CAN render Client Components                       │
│    • CAN pass Server Components as children/props to    │
│      Client Components (composition pattern)            │
│                                                         │
│    ┌─────────────────────────────────────────────────┐  │
│    │  Client Component  ("use client" at top)       │  │
│    │    • useState, useEffect, event handlers       │  │
│    │    • Zustand hooks, browser APIs               │  │
│    │    • CANNOT import server-only modules         │  │
│    │    • CANNOT be async (except in rare cases)    │  │
│    │                                                 │  │
│    │    {children}  ← can be a Server Component     │  │
│    │                  passed from the parent        │  │
│    └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Rule of thumb:** push `"use client"` as far down the tree as physically possible. A button that needs `onClick` becomes a client component — its parent stays on the server.

**Composition trick — learn this, it solves 80% of "I need client state but also server data" problems:**

```tsx
// Server Component — fetches data
export default async function DashboardPage() {
  const stats = await getDashboardStats();
  return (
    <ClientShell> {/* client component with a sidebar toggle */}
      <StatsGrid stats={stats} /> {/* server component passed as children */}
    </ClientShell>
  );
}
```

`ClientShell` never imports `StatsGrid` — it receives it as `children`. The server component stays server-side. The client boundary stays tiny.

---

## The static-shell contract (the most important rule in this file)

**The rule:** `page.tsx` renders a **static layout shell** instantly. Any component that needs data is wrapped in `<Suspense fallback={<XSkeleton />}>` and does its own fetching.

### ✅ Correct pattern

```tsx
// app/(app)/dashboard/page.tsx
import { Suspense } from "react";
import { PageHeader } from "@/components/common/page-header";
import { RevenueCard } from "./_components/revenue-card";
import { RevenueCardSkeleton } from "./_components/revenue-card.skeleton";
import { RecentOrders } from "./_components/recent-orders";
import { RecentOrdersSkeleton } from "./_components/recent-orders.skeleton";

// NOTE: page.tsx itself is NOT async. It fetches nothing.
export default function DashboardPage() {
  return (
    <div className="flex flex-col gap-6 p-6">
      <PageHeader title="Dashboard" description="Your business at a glance" />

      <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
        <Suspense fallback={<RevenueCardSkeleton />}>
          <RevenueCard />
        </Suspense>
        {/* ...other cards, each in its own Suspense */}
      </div>

      <Suspense fallback={<RecentOrdersSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}
```

```tsx
// app/(app)/dashboard/_components/revenue-card.tsx
import { getRevenueSummary } from "@/lib/db/queries/revenue";
import { createClient } from "@/lib/supabase/server";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export async function RevenueCard() {
  const supabase = await createClient();       // per-request client, see 20-supabase.md
  const data = await getRevenueSummary(supabase);
  return (
    <Card>
      <CardHeader><CardTitle>Revenue</CardTitle></CardHeader>
      <CardContent>
        <div className="text-3xl font-bold">${data.total.toLocaleString()}</div>
        <p className="text-sm text-muted-foreground">{data.deltaLabel}</p>
      </CardContent>
    </Card>
  );
}
```

```tsx
// app/(app)/dashboard/_components/revenue-card.skeleton.tsx
import { Card, CardContent, CardHeader } from "@/components/ui/card";
import { Skeleton } from "@/components/ui/skeleton";

export function RevenueCardSkeleton() {
  return (
    <Card>
      <CardHeader><Skeleton className="h-5 w-24" /></CardHeader>
      <CardContent className="space-y-2">
        <Skeleton className="h-8 w-32" />
        <Skeleton className="h-4 w-40" />
      </CardContent>
    </Card>
  );
}
```

### ❌ Forbidden patterns

```tsx
// ❌ page.tsx is async and blocks on data
export default async function DashboardPage() {
  const stats = await getStats();          // blocks the entire shell
  const orders = await getOrders();        // waterfall — blocks again
  return <Dashboard stats={stats} orders={orders} />;
}

// ❌ fetching inside useEffect in a client component
"use client";
export function RevenueCard() {
  const [data, setData] = useState(null);
  useEffect(() => { fetch("/api/revenue").then(r => r.json()).then(setData); }, []);
  // NO. Render waterfall + no caching + no streaming + no SSR.
}

// ❌ skeleton rendered via conditional inside the component
export async function RevenueCard() {
  const data = await getData();
  if (!data) return <Skeleton />; // skeleton belongs in the Suspense fallback, not here
}
```

### Skeleton naming rule

Every component that does I/O ships with a sibling file:

```
revenue-card.tsx
revenue-card.skeleton.tsx   ← exports RevenueCardSkeleton
```

Non-negotiable. LLMs will otherwise re-implement the skeleton three times with three different widths.

---

## Caching / revalidation

- **Default:** let Next cache fetches indefinitely.
- **Time-based:** `fetch(url, { next: { revalidate: 60 } })` or `export const revalidate = 60` on the route.
- **Force fresh** (authenticated user pages): `export const dynamic = "force-dynamic"` — **required** on any route that reads session cookies.
- **On-demand:** `revalidateTag("orders")` / `revalidatePath("/dashboard")` from a Server Action.

**Auth + ISR warning:** Never apply ISR (`revalidate`) to a route that reads a user session. Cached HTML will be served with another user's `Set-Cookie` header and you'll log people in as each other. See `20-supabase.md` for the full story.

---

## Error & loading hierarchy

Every route group has an `error.tsx` (client component with a reset button) and, where relevant, a `not-found.tsx`. These are the **only** place error UI gets designed — individual components should throw and let the boundary handle it.

```tsx
// app/(app)/dashboard/error.tsx
"use client";
import { Button } from "@/components/ui/button";

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div className="flex flex-col items-center justify-center gap-4 p-12">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      <p className="text-sm text-muted-foreground">{error.message}</p>
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

**Loading priority:**
1. **Preferred:** per-component `<Suspense fallback={<XSkeleton />}>`.
2. **Fallback:** route-level `loading.tsx` for the whole segment — use when every component on the page depends on the same data.
3. **Last resort:** inline `isLoading` state — only in forms during mutation, never for initial render.

**Empty states are a real UI, not an afterthought.** `components/common/empty-state.tsx` is a reusable component with an icon, title, description, and optional CTA. Every list/table uses it.

---

## Server Actions — the mutation path

All mutations go through Server Actions. Not API routes. Not client-side `fetch`.

```ts
// lib/db/mutations/orders.ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { createOrderSchema } from "@/lib/validators/order";
import type { ActionResult } from "@/types";

export async function createOrder(
  _prev: ActionResult,
  formData: FormData
): Promise<ActionResult> {
  const parsed = createOrderSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };
  }

  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return { ok: false, error: "Unauthorized" };

  const { error } = await supabase.from("orders").insert({
    ...parsed.data,
    user_id: user.id,
  });
  if (error) return { ok: false, error: error.message };

  revalidatePath("/dashboard/orders");
  redirect("/dashboard/orders");
}
```

```tsx
// client form
"use client";
import { useActionState } from "react";
import { createOrder } from "@/lib/db/mutations/orders";

export function CreateOrderForm() {
  const [state, action, pending] = useActionState(createOrder, { ok: true });
  return (
    <form action={action} className="space-y-4">
      {/* inputs */}
      <Button disabled={pending}>{pending ? "Creating…" : "Create"}</Button>
      {!state.ok && state.error && <p className="text-destructive">{state.error}</p>}
    </form>
  );
}
```

**`ActionResult` is declared once in `src/types/index.ts`:**

```ts
export type ActionResult<T = void> =
  | { ok: true; data?: T }
  | { ok: false; error?: string; fieldErrors?: Record<string, string[]> };
```

Every action returns this shape. No exceptions. No bespoke result types per action.

**Zod schemas are shared** between the Server Action (`.safeParse`) and the client form (`zodResolver`). One schema, two uses, zero drift. See `90-discipline.md` for the single-source-of-truth rules.
