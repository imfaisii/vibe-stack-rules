# Verify — Phase 5

> Last phase. Confirms the generated code compiles, lints, and — when the adapter is active — satisfies the vibe-stack-rules pre-submit checklist.
>
> **Do not report success to the user until every mandatory check passes.**

---

## 5.1 Detect the package manager

Before running any command, detect which package manager the project uses. Check in this order:

| File present | Package manager | Run commands with |
|---|---|---|
| `pnpm-lock.yaml` | pnpm | `pnpm <script>` |
| `yarn.lock` | yarn | `yarn <script>` |
| `bun.lockb` or `bun.lock` | bun | `bun run <script>` |
| `package-lock.json` (only) | npm | `npm run <script>` |

If no lockfile, ask the user which to use. Do not guess.

Also check which scripts exist in `package.json`. If `typecheck` / `lint` aren't defined, fall back to `npx tsc --noEmit` and `npx eslint .` respectively — and flag to the user that the scripts are missing so they can add them.

---

## 5.2 Mandatory checks (run in order, stop on first failure)

### Check 1 — TypeScript compiles

```bash
<pm> typecheck     # or: npx tsc --noEmit
```

Must exit 0. Any error = fix it and re-run. Do not "work around" a type error with `any` or `as unknown as Foo`.

Common failure modes to fix in place:

| Error | Fix |
|---|---|
| `Cannot find module '@/components/ui/<x>'` | The user hasn't run `shadcn add <x>` yet. Re-emit the install command. |
| `Property 'X' does not exist on type 'Y'` | Fix the prop type, not the consumer. |
| `Module '"next/image"' has no exported member 'Image'` | You used `import { Image }` — it's a default import. Fix. |
| Strict-mode errors on `noUncheckedIndexedAccess` | Add the missing undefined handling. Don't disable the rule. |

### Check 2 — ESLint clean

```bash
<pm> lint          # or: npx eslint .
```

Must exit 0. Warnings are allowed but every error must be fixed.

### Check 3 — No forbidden patterns in generated files

Grep the diff for these and flag any hits:

| Pattern | Why it's wrong |
|---|---|
| `: any` / `as any` | `any` is banned — use `unknown` + type guard. |
| `style={{ background` / `color:` with `#` | Hex in JSX — use Tailwind tokens. |
| `className="..."\+...` or template literals for classes | Must go through `cn()`. |
| `import .* from ['"]@supabase/auth-helpers` | Deprecated — use `@supabase/ssr`. |
| `href="/[a-z].*"` in adapter mode | Hard-coded route string — use `ROUTES.*`. |
| `import ['"].*\.css['"]` in a `.tsx` file | Mode A forbids source CSS imports. Mode B allows only `*.module.css`. Mode C permits plain `.css` imports. |

### Style-mode conformance grep (run based on Phase 1.5 decision)

```bash
# Mode A: no .css imports anywhere except app/globals.css
grep -rnE "import\s+['\"].*\.css['\"]" app/ components/ 2>/dev/null \
  | grep -v "globals.css" \
  && echo "FAIL: Mode A should have no CSS imports other than globals.css"

# Modes A + B: flag source CSS class names that look untranslated.
# Heuristic: kebab-case classNames that are not recognized Tailwind utilities.
grep -rnE 'className=["\x27][a-z]+-[a-z]+["\x27]' app/ components/ 2>/dev/null \
  | grep -vE '(flex-|grid-|gap-|px-|py-|text-|bg-|border-|rounded-|shadow-|w-|h-|min-|max-|space-|items-|justify-)' \
  && echo "WARN: possible untranslated source class names in JSX — spot-check these"
```

The second grep is a warning-level signal, not a hard fail — it can false-positive on legitimate domain class names (e.g., `data-grid`, `user-avatar`) the extractor kept. The user eyeballs the output.

---

## 5.3 Adapter-mode additional checks (vibe-stack-rules only)

Run these only when `adapter = "vibe-stack-rules"`.

### Check 4 — Pre-submit checklist from stack/90-discipline.md

Reproduced here so the skill is self-contained:

- [ ] No `await` in `page.tsx`
- [ ] No `fetch` / `supabase.from` inside `useEffect`
- [ ] No Supabase client at module scope
- [ ] No hard-coded route strings (use `ROUTES`)
- [ ] No duplicate zod schemas (import from `lib/validators/`)
- [ ] No duplicate database types (import from `types/database.ts`)
- [ ] No missing Skeleton sibling for any data-fetching component
- [ ] No component larger than ~150 lines
- [ ] No `useState` for data that came from the server (it's a prop)
- [ ] No `any`
- [ ] No ISR / `force-static` on authenticated routes
- [ ] `export const dynamic = "force-dynamic"` on any route reading session
- [ ] All shadcn components composed via `cn()` for className
- [ ] All mutations go through Server Actions returning `ActionResult`
- [ ] Zustand stores (if any) wrapped in Provider, created per-request, selectors used

Run the grep checks below for the mechanically-verifiable items, and call out the rest for manual review.

```bash
# page.tsx must not be async
grep -rn "^export default async function.*Page" app/ && echo "FAIL: async page.tsx"

# Every data-fetching component should have a sibling .skeleton.tsx
# (heuristic — list components that are async and check)
for f in $(grep -rln "^export async function" app/ components/); do
  base="${f%.tsx}"
  [ ! -f "${base}.skeleton.tsx" ] && echo "MISSING: ${base}.skeleton.tsx"
done

# No supabase client at module scope (should always be inside a function)
grep -rn "^const supabase = create" app/ components/ lib/ && echo "FAIL: module-scope supabase client"

# No hard-coded route strings
grep -rnE '(href|redirect\()\s*=?\s*["'"'"']\/[a-z]' app/ components/ && echo "FAIL: hard-coded route"
```

Report each failure with the file + line + fix suggestion. Do not auto-fix without telling the user which files you touched.

---

## 5.4 Report format

After all checks, emit a single summary block:

```markdown
## Verification report

**Adapter:** vibe-stack-rules | none
**Package manager:** pnpm | npm | yarn | bun

### Checks
- [x] TypeScript compiles (tsc --noEmit)
- [x] ESLint clean
- [x] No forbidden patterns
- [x] Pre-submit checklist (adapter mode)

### Files created
- components/common/stat-card.tsx
- app/(marketing)/landing/_components/hero-section.tsx
- app/(marketing)/landing/_components/hero-section.skeleton.tsx
- app/(marketing)/landing/page.tsx
- ...

### Follow-up for you
1. Run the shadcn install:
   ```
   npx shadcn@latest add button card skeleton ...
   ```
2. Re-run `<pm> typecheck` after installing — the missing `@/components/ui/*` errors will resolve.
3. Wire the Server Action stubs in `lib/db/mutations/*` to real queries.
4. Review the hero-section copy — I kept the source strings verbatim; they may need your editorial pass.

### Manual review items (not mechanically verifiable)
- Does the ROUTES mapping match your existing routing conventions?
- Are the extracted component names (StatCard, etc.) consistent with your codebase's naming?
```

If any mandatory check failed, lead with the failure, show the user the exact command output, and propose a fix. Do not mark the task done.

---

## 5.5 Exit

On green: hand back to the user with the report above and stop.
On red: fix the failure, re-run the failing check, and continue. Loop until green.
