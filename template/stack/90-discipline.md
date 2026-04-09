# Stack Rules — Discipline

> Covers: killing redundant declarations, TypeScript rules, the LLM pre-submit checklist, and the anti-patterns cheatsheet.

---

## Redundant declarations — how we kill them

Redundant declarations are the single biggest scaling failure in LLM-maintained codebases. The rule is simple:

> **If a string literal, type, shape, constant, route, or query appears more than once, one of the copies is a bug.**

### Single source of truth table

| Thing | Lives in | Imported by |
|---|---|---|
| Database row types | `types/database.ts` (generated) | Queries, Server Actions, anything that touches DB |
| Domain/input schemas | `lib/validators/*.ts` (zod) | Server Actions + client forms |
| Route paths | `lib/constants.ts` → `ROUTES` | `<Link>`, `redirect()`, `router.push()` — **never** hard-coded strings |
| Enum values (statuses, roles) | `lib/constants.ts` or a zod enum in `lib/validators/` | Everywhere |
| Supabase queries | `lib/db/queries/*` | Server components + Server Actions only |
| `cn` helper | `lib/utils.ts` | Every component that combines classes |
| `ActionResult` type | `types/index.ts` | Every Server Action |
| UI state (sidebar, modals) | `stores/*-store.ts` | Client components via provider |

### Example: killing route-string duplication

```ts
// lib/constants.ts
export const ROUTES = {
  home: "/",
  login: "/login",
  dashboard: "/dashboard",
  orders: "/dashboard/orders",
  orderDetail: (id: string) => `/dashboard/orders/${id}`,
} as const;
```

```tsx
// ✅
import { ROUTES } from "@/lib/constants";
<Link href={ROUTES.orders}>Orders</Link>
redirect(ROUTES.login);

// ❌
<Link href="/dashboard/orders">Orders</Link>  // typo-prone, rename-hostile
```

### LLM self-check before declaring anything

Before declaring a new type, constant, helper, or component, do this:

1. `grep` the repo for the name and for the shape.
2. If something similar exists, **import it** or **extend it** — never clone it.
3. If the existing version is slightly wrong, **fix it in place** and update all callers. Don't ship a parallel version.
4. If two things look identical but must diverge, give them distinct names that explain why they diverge.

---

## TypeScript rules

- `strict: true`, `noUncheckedIndexedAccess: true`, `noImplicitOverride: true`.
- **No `any`.** Ever. If you're tempted, you want `unknown` + a type guard.
- **No type assertions** (`as Foo`) except at trust boundaries (parsed JSON, third-party libs). Use zod or a type guard.
- **Props go in an exported `type` next to the component** — not inline on the function signature past 2 props:

  ```tsx
  type RevenueCardProps = { initialRange?: DateRange; className?: string };
  export function RevenueCard({ initialRange, className }: RevenueCardProps) { ... }
  ```
- Import types with `import type { ... }` to keep runtime bundles clean.

---

## LLM rules of engagement

When asked to add a feature, follow this sequence:

1. **Read the file(s) involved** and the adjacent `_components/` folder.
2. **Check `lib/db/queries/` and `lib/validators/`** for existing functions/schemas that cover the domain. If one exists, use it.
3. **Check `stores/`** for existing client state that covers the domain.
4. **Identify the RSC boundary** — which part is server, which is client — before writing a single line.
5. **Write the server component first**, then the client islands it needs.
6. **Write the skeleton file** before writing the async component.
7. **Wire the Suspense boundary in `page.tsx`**, keeping `page.tsx` non-async.
8. **Never change a shadcn primitive in `components/ui/`.** Wrap it instead.
9. **Never add a new top-level dependency** without telling the user first.
10. **After finishing:** scan the diff for duplicated types, duplicated route strings, new `any`s, new `useEffect` fetches, and new module-scope Supabase clients. Fix them before claiming done.

### Pre-submit checklist (run this against your own diff before claiming done)

- [ ] No `await` in `page.tsx`
- [ ] No `fetch` / `supabase.from` inside `useEffect`
- [ ] No Supabase client at module scope
- [ ] No hard-coded route strings (use `ROUTES`)
- [ ] No duplicate zod schemas (import from `lib/validators/`)
- [ ] No duplicate database types (import from `types/database.ts`)
- [ ] No missing Skeleton sibling for any data-fetching component
- [ ] No component larger than ~150 lines (if larger, split it)
- [ ] No `useState` for data that came from the server (it's a prop)
- [ ] No `any`
- [ ] No ISR / `force-static` on authenticated routes
- [ ] `export const dynamic = "force-dynamic"` on any route reading session
- [ ] All shadcn components used via `cn()` for className composition
- [ ] All mutations go through Server Actions returning `ActionResult`
- [ ] Zustand stores wrapped in a Provider, created per-request, selectors used

---

## Anti-patterns cheatsheet (quick reference)

| Anti-pattern | Fix |
|---|---|
| `async function Page()` that awaits data | Static shell + `<Suspense>` around data components |
| `useEffect(() => fetch(...))` | Move to Server Component, or a Server Action on interaction |
| Supabase client declared at top of file | Move inside the function, per-request |
| `useSession()` / `getSession()` for authz | `supabase.auth.getUser()` |
| `<Card>` tree repeated 4 times | Extract `<StatCard>` in `components/common/` |
| Two zod schemas for the same shape | One in `lib/validators/`, import both places |
| `"/dashboard/orders"` hard-coded | `ROUTES.orders` |
| `create(store)` at module scope in Next.js | Store Factory + Provider pattern |
| Skeleton rendered conditionally inside component | Sibling `.skeleton.tsx` + Suspense fallback |
| `useStore()` without a selector | `useStore((s) => s.field)` |
| Route handler for a simple form submit | Server Action |
| `as Foo` to silence a type error | Fix the types, or use a zod parse |
| `any` anywhere | `unknown` + type guard, or a proper type |
| Hex color in JSX | Tailwind token (`bg-primary`, etc.) |

---

*End of discipline rules. If you're an LLM reading this: acknowledge these files by respecting them in the code you write. That is the only acknowledgement that matters.*
