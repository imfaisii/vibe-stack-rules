# Stack Rules — Client-Side: State and UI

> Covers: Zustand (what it is and isn't for, the store factory pattern) and shadcn/ui (composition, `cn()`, theming).

---

## Part 1 — Zustand

### What Zustand is for

- Open/close state for sheets, modals, command menus
- Theme, sidebar collapse, layout density
- Multi-step wizard progress, in-progress form drafts
- Optimistic UI flags, local filter/sort state, selection sets
- Cross-component client state that Context would make ugly

### What Zustand is **NOT** for

- ❌ Cached API responses / server data (use RSC + Suspense, or TanStack Query if the project needs it)
- ❌ The current user (read from Supabase in a Server Component and pass down, or read once and stash in a per-request store)
- ❌ Anything that must survive a page reload and can live in a cookie or the URL (use URL state / cookies)
- ❌ Anything needed during SSR rendering (it won't be there)

### The store-per-request pattern (mandatory)

A single global `create(...)` at module scope is **shared across server requests** on Next.js's server runtime → user A's data leaks to user B. We use the **Store Factory + Provider** pattern from the Zustand docs.

```ts
// stores/ui-store.ts
import { createStore } from "zustand/vanilla";

export type UiState = {
  sidebarOpen: boolean;
  commandMenuOpen: boolean;
};

export type UiActions = {
  toggleSidebar: () => void;
  setSidebar: (open: boolean) => void;
  setCommandMenu: (open: boolean) => void;
};

export type UiStore = UiState & UiActions;

export const defaultUiState: UiState = {
  sidebarOpen: true,
  commandMenuOpen: false,
};

export const createUiStore = (init: UiState = defaultUiState) =>
  createStore<UiStore>()((set) => ({
    ...init,
    toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
    setSidebar: (open) => set({ sidebarOpen: open }),
    setCommandMenu: (open) => set({ commandMenuOpen: open }),
  }));
```

```tsx
// stores/ui-store-provider.tsx
"use client";
import { createContext, useContext, useRef, type ReactNode } from "react";
import { useStore } from "zustand";
import { createUiStore, type UiStore } from "./ui-store";
import type { StoreApi } from "zustand/vanilla";

const UiStoreContext = createContext<StoreApi<UiStore> | null>(null);

export function UiStoreProvider({ children }: { children: ReactNode }) {
  const storeRef = useRef<StoreApi<UiStore> | null>(null);
  if (!storeRef.current) storeRef.current = createUiStore();
  return (
    <UiStoreContext.Provider value={storeRef.current}>
      {children}
    </UiStoreContext.Provider>
  );
}

export function useUiStore<T>(selector: (s: UiStore) => T): T {
  const store = useContext(UiStoreContext);
  if (!store) throw new Error("useUiStore must be used within UiStoreProvider");
  return useStore(store, selector);
}
```

```tsx
// app/layout.tsx (or a client Providers wrapper)
import { UiStoreProvider } from "@/stores/ui-store-provider";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <UiStoreProvider>{children}</UiStoreProvider>
      </body>
    </html>
  );
}
```

```tsx
// consumer
"use client";
import { useUiStore } from "@/stores/ui-store-provider";

export function SidebarToggle() {
  const toggle = useUiStore((s) => s.toggleSidebar);
  return <Button onClick={toggle}>Toggle</Button>;
}
```

**Selectors are mandatory.** Always `useUiStore((s) => s.x)` — never `useUiStore()` without a selector, which subscribes to the entire store and re-renders on every change.

### Persistence

Use `zustand/middleware` `persist` **only for stores that should survive reloads** (theme, sidebar state). Wrap the consumer in a "wait for hydration" hook to avoid SSR/CSR mismatches:

```ts
// hooks/use-hydrated.ts
"use client";
import { useEffect, useState } from "react";
export function useHydrated() {
  const [h, setH] = useState(false);
  useEffect(() => setH(true), []);
  return h;
}
```

```tsx
const hydrated = useHydrated();
const sidebarOpen = useUiStore((s) => s.sidebarOpen);
if (!hydrated) return <SidebarSkeleton />;
```

---

## Part 2 — shadcn/ui + Tailwind

### Installing components

```bash
npx shadcn@latest add button card dialog skeleton form input ...
```

- **Never** hand-write a shadcn primitive from memory. Always install via CLI.
- **Never** edit files in `components/ui/` unless you're extending a primitive — and even then, prefer a wrapper in `components/common/`.

### The `cn()` helper is the only way to combine classes

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// ✅
<Button className={cn("w-full", isDanger && "bg-destructive")} />

// ❌ string concatenation → can't override, tailwind-merge can't dedupe
<Button className={`w-full ${isDanger ? "bg-destructive" : ""}`} />
```

### Theming

- Colors come from CSS variables in `app/globals.css` (set up by shadcn). **Never hard-code hex values in JSX.** Use `bg-primary`, `text-muted-foreground`, etc.
- Dark mode is handled by `next-themes`. Every component must look correct in both themes.
- Spacing: use the Tailwind scale. No arbitrary pixel values (`p-[13px]`) unless you can justify it in a comment.

### Composition pattern — wrap shadcn primitives in domain components

```tsx
// components/common/stat-card.tsx — domain-specific wrapper
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { cn } from "@/lib/utils";

type StatCardProps = {
  title: string;
  value: string | number;
  delta?: string;
  className?: string;
};

export function StatCard({ title, value, delta, className }: StatCardProps) {
  return (
    <Card className={cn("", className)}>
      <CardHeader>
        <CardTitle className="text-sm font-medium text-muted-foreground">
          {title}
        </CardTitle>
      </CardHeader>
      <CardContent>
        <div className="text-3xl font-bold">{value}</div>
        {delta && <p className="text-xs text-muted-foreground">{delta}</p>}
      </CardContent>
    </Card>
  );
}
```

Now every dashboard metric uses `<StatCard>` — not three different `<Card>` + `<CardHeader>` + `<CardContent>` trees. **If you find yourself repeating a shadcn tree more than twice, it becomes a domain component.**

### Forms

Forms use shadcn's `Form` + `react-hook-form` + `zod`. The zod schema lives in `lib/validators/` and is **the same schema used by the Server Action** for server-side validation. One schema, two uses, zero drift.

```ts
// lib/validators/order.ts
import { z } from "zod";
export const createOrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.coerce.number().int().min(1).max(999),
  notes: z.string().max(500).optional(),
});
export type CreateOrderInput = z.infer<typeof createOrderSchema>;
```

This schema is imported by:
1. The Server Action (see `10-server.md`)
2. The client form's `zodResolver`
3. Anywhere else that needs the `CreateOrderInput` type

**One declaration. Three uses. Zero redundancy.**
