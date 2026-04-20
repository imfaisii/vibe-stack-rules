# Synthesize + Generate — Phases 3 & 4

> Phase 3 merges the three analyze-phase outputs into a single plan and waits for user approval.
> Phase 4 writes files in strict dependency order.
>
> **If the vibe-stack-rules adapter is active, also load `adapter-vibe-stack-rules.md` and apply its additional rules.**

---

## Phase 3 — Synthesize

### 3.1 Merge the three analyses

Combine Style Mapper + Component Extractor + shadcn Matcher outputs into one plan. Apply these resolution rules in order:

1. **shadcn wins naming conflicts.** If Style Mapper produced `colors.brand` and shadcn Matcher expects `primary`, rename to `primary` and add `brand` as an alias if the design references it.
2. **Extractor wins over inline.** If Component Extractor found a 3+ repeating pattern that shadcn Matcher mapped to raw `<Card>...<CardHeader>...` trees, wrap the shadcn tree inside the extracted component. The extracted component becomes the import target; raw shadcn usage only appears inside it.
3. **Dedupe shadcn install list.** Union of primitives used by Matcher + primitives referenced by Extractor. One `shadcn add` line.
4. **Dependency order for extractions.** Topologically sort — a component referenced by other components is written first.

### 3.2 Produce the conversion plan

Format the plan as a single markdown block with this structure:

```markdown
## Conversion plan for <target file>

### shadcn primitives to install
```bash
npx shadcn@latest add <space-separated list>
```

### Tailwind config changes
- Add color tokens: `primary: ...`, `muted: ...`, ...
- Add font family: `sans: ["Inter", ...]`
- Add plugin: `@tailwindcss/typography` (if the design uses long-form prose)
(Omit sections that have no changes.)

### Reusable components to extract
| Name | Path | Occurrences | Why |
|---|---|---|---|
| StatCard | components/common/stat-card.tsx | 4 | Dashboard metric tiles |
| ...      | ...                              | ... | ... |

### Files to create (in write order)
1. tailwind.config.ts  (updated)
2. app/globals.css     (CSS variable updates)
3. components/common/stat-card.tsx
4. app/(marketing)/landing/_components/hero-section.tsx
5. app/(marketing)/landing/_components/feature-grid.tsx
6. app/(marketing)/landing/page.tsx
7. ...

### Adapter layer (if active)
- Skeleton siblings for: FeatureGrid, TestimonialCarousel
- Suspense boundaries: landing page wraps FeatureGrid and TestimonialCarousel each in Suspense
- ROUTES entries added: landing = "/"

### Open questions
- <anything ambiguous from the analyze phase>
```

### 3.3 Approval checkpoint — mandatory

After showing the plan, **stop and wait** for the user to approve, edit, or reject. Do not write any files in Phase 4 until one of:

- Explicit approval ("yes", "go", "proceed", "looks good")
- An edited plan (user changed something — re-show and wait again)

If the user approves, print a one-line summary of what's about to happen and move to Phase 4.

---

## Phase 4 — Generate

### 4.1 Write order (strict)

Files are written in this order — never break the sequence:

```
1.  lib/utils.ts                            (create cn() if missing; otherwise skip)
2.  lib/constants.ts                        (add ROUTES entries — adapter only)
3.  tailwind.config.ts                      (token + theme extensions)
4.  app/globals.css                         (CSS variable additions; no hex in JSX ever)
5.  shadcn install command                  (emit for user to run; do NOT attempt to run it)
6.  components/common/*.tsx                 (extracted shared components)
7.  app/<route>/_components/*.tsx           (route-private components)
8.  app/<route>/_components/*.skeleton.tsx  (adapter only — sibling per data-fetching component)
9.  app/<route>/page.tsx                    (wires Suspense + imports; non-async if adapter on)
10. app/<route>/error.tsx                   (if the design includes error state)
11. app/<route>/loading.tsx                 (optional, prefer per-component Suspense)
```

**If any file depends on something not yet written, move it later.** Never emit broken imports.

### 4.2 Universal generation rules (apply in all modes)

| Rule | Detail |
|---|---|
| **TypeScript** | `.tsx` for components, `.ts` for utilities. Strict mode assumed. No `any`. |
| **Props** | Exported `type <Name>Props = { ... }` above the component. Accept `className?: string` for composition. |
| **className composition** | Always `cn(...)` from `@/lib/utils`. Never string concat. |
| **No inline styles** | `style={{ ... }}` is banned except for truly dynamic values (e.g. a percentage width driven by data). |
| **No hex in JSX** | Use Tailwind tokens (`bg-primary`, `text-muted-foreground`). |
| **Imports** | `@/` path alias for internal. `import type` for type-only imports. |
| **Server/client split** | Default to server. Add `"use client"` only when the component uses `useState`, `useEffect`, event handlers, or browser APIs. |
| **Accessibility** | `aria-*` preserved from the source HTML. `<button>` for actions, `<a>` (or `<Link>`) for navigation. `alt=""` on decorative images. |
| **Images** | `next/image` with `width`, `height`, `alt`. Use `priority` for above-the-fold. |
| **Links** | `next/link` for internal. `<a target="_blank" rel="noopener noreferrer">` for external. |

### 4.3 Component template (vanilla, no adapter)

```tsx
// components/common/stat-card.tsx
import { cn } from "@/lib/utils";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export type StatCardProps = {
  title: string;
  value: string | number;
  delta?: string;
  className?: string;
};

export function StatCard({ title, value, delta, className }: StatCardProps) {
  return (
    <Card className={cn(className)}>
      <CardHeader>
        <CardTitle className="text-sm font-medium text-muted-foreground">
          {title}
        </CardTitle>
      </CardHeader>
      <CardContent>
        <div className="text-3xl font-bold">{value}</div>
        {delta ? <p className="text-xs text-muted-foreground">{delta}</p> : null}
      </CardContent>
    </Card>
  );
}
```

### 4.4 Page template (vanilla, no adapter)

```tsx
// app/(marketing)/landing/page.tsx
import { HeroSection } from "./_components/hero-section";
import { FeatureGrid } from "./_components/feature-grid";

export default function LandingPage() {
  return (
    <main className="mx-auto flex max-w-6xl flex-col gap-16 px-6 py-12">
      <HeroSection />
      <FeatureGrid />
    </main>
  );
}
```

### 4.5 When the adapter is active

Apply additional rules from `adapter-vibe-stack-rules.md`. Key deltas:

- `page.tsx` is **never** `async`; data components are each wrapped in `<Suspense fallback={<XSkeleton />}>`.
- Every component that fetches data gets a sibling `*.skeleton.tsx`.
- `"use client"` is pushed as deep as possible — favor the composition pattern (client shell receives server children).
- Mutations emit Server Action **stubs** (not full implementations) returning `ActionResult` — the user wires the actual DB logic.
- Route strings go through `ROUTES` from `@/lib/constants`.

### 4.6 Server Action stub shape (adapter only)

When the design implies a form, emit a stub like:

```ts
// lib/db/mutations/<domain>.ts
"use server";
import type { ActionResult } from "@/types";
import { <schemaName> } from "@/lib/validators/<domain>";

export async function <actionName>(
  _prev: ActionResult,
  formData: FormData
): Promise<ActionResult> {
  const parsed = <schemaName>.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };
  }

  // TODO: wire DB / auth here.
  // Example:
  //   const supabase = await createClient();
  //   const { error } = await supabase.from("<table>").insert(parsed.data);
  //   if (error) return { ok: false, error: error.message };

  return { ok: true };
}
```

And the matching zod schema in `lib/validators/<domain>.ts` — **the same schema** is used by the client form's `zodResolver`.

### 4.7 What NOT to do in generation

- Do **not** attempt to run `shadcn add`. Emit the command and tell the user to run it.
- Do **not** infer business logic. If the design shows a pricing table with dollar amounts, put those in a `PRICING` constant in `lib/constants.ts` — don't wire a real API.
- Do **not** invent routes that aren't in the design. If only `landing.html` is being implemented, don't also scaffold `/dashboard`.
- Do **not** edit files in `components/ui/` if they already exist. shadcn primitives are install-only.
- Do **not** add dependencies beyond what the shadcn install requires. No `@emotion/*`, no `clsx` (it comes with `cn`), no `framer-motion` unless the design genuinely needs animation and then ask first.

---

## Phase 3 → 4 handoff summary

```
Synthesizer completes plan
       ↓
Show plan to user
       ↓
Wait for approval ← mandatory checkpoint
       ↓
Generator writes files in strict order
       ↓
Hand off to Phase 5 (verify)
```

Move to `verify.md`.
