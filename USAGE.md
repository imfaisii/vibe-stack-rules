# Using the `claude-design-to-nextjs` skill

> End-to-end walkthrough: from a Claude Design URL to shipped Next.js components.

---

## What this skill does

You open [claude.ai/design](https://claude.ai/design), describe a UI, and get a design back along with a standardized implementation command that looks like:

```
Fetch this design file, read its readme, and implement the relevant aspects of the design.
https://api.anthropic.com/v1/design/h/<id>?open_file=landing.html
Implement: landing.html
```

Paste that into Claude Code in your project. The skill auto-triggers, fetches the bundle (HTML + CSS + components + README), and converts it into **production Next.js App Router code** — TypeScript, Tailwind, shadcn/ui — using a 5-phase pipeline with parallel analysis subagents and a mandatory human approval checkpoint.

---

## Prerequisites

Before the first run, make sure your project has:

- **Next.js 15+ App Router** (the skill does not target the Pages Router).
- **Tailwind CSS** configured. If not: follow the Tailwind + Next.js setup doc.
- **shadcn/ui initialized** — run once per project:
  ```bash
  npx shadcn@latest init
  ```
- **`vibe-stack-rules` installed** (optional but recommended — unlocks the adapter layer):
  ```bash
  npx degit imfaisii/vibe-stack-rules/template .
  ```
  Once `stack/*.md` is present at repo root, the skill automatically layers in Suspense + skeleton siblings + non-async `page.tsx` + Server Action stubs + `ROUTES` constants + `cn()` composition. Without it, the skill emits clean vanilla Next.js code.

Also install the skill itself — it ships inside `template/skills/`, so the same `degit` command above brings it along. Verify:

```bash
ls skills/claude-design-to-nextjs/SKILL.md
```

---

## The happy path — 5 steps

### 1. Create your design in Claude Design

Describe the UI you want ("a pricing page with three tiers, a FAQ, and a CTA banner"). Claude Design produces a bundle of HTML + CSS + component files and hands you an implementation command.

### 2. Paste the command into Claude Code

Copy the full command (both lines — URL *and* `Implement:`) and paste it into Claude Code as your message. The skill's frontmatter description matches this shape, so Claude Code will invoke the skill without any extra flag.

You should see Claude announce one of:

```
Detected vibe-stack-rules contract at ./stack/. Applying adapter layer.
```

or

```
No vibe-stack-rules stack/ folder found. Running in vanilla mode.
```

### 3. Review the conversion plan (mandatory approval checkpoint)

After the 3 parallel analysis subagents return (~30-60 seconds), Claude shows you a plan block:

```markdown
## Conversion plan for landing.html

### shadcn primitives to install
```bash
npx shadcn@latest add button card skeleton badge ...
```

### Tailwind config changes
- Add color tokens: primary, muted, accent
- Add font family: sans → ["Inter", ...]

### Reusable components to extract
| Name          | Path                                  | Occurrences | Why                         |
|---------------|---------------------------------------|-------------|-----------------------------|
| PricingTile   | components/common/pricing-tile.tsx    | 3           | Pricing page tiers          |
| FeatureRow    | components/common/feature-row.tsx     | 4           | Feature comparison list     |

### Files to create (in write order)
1. tailwind.config.ts  (updated)
2. app/globals.css     (CSS variable updates)
3. components/common/pricing-tile.tsx
4. components/common/feature-row.tsx
5. app/(marketing)/landing/_components/hero-section.tsx
6. app/(marketing)/landing/_components/hero-section.skeleton.tsx
7. app/(marketing)/landing/page.tsx

### Open questions
- The design includes a newsletter form — should I wire this as a Server Action stub, or skip it?
```

**This is your approval checkpoint.** Read the plan. Say:

- **"go" / "approve" / "looks good"** — files get written in the order shown.
- **Edit the plan** ("skip the newsletter form, rename PricingTile to PlanCard") — Claude updates and re-presents until you approve.
- **"stop"** — no files are written. Clean exit.

### 4. Run the shadcn install

The skill **does not** run `shadcn add` for you (by design — it's a filesystem-modifying command that installs primitives into your `components/ui/` folder). After approval, Claude writes the non-shadcn files first, then reminds you to run:

```bash
npx shadcn@latest add button card skeleton badge ...
```

Run that in a terminal. The `@/components/ui/*` import errors will resolve once the primitives are installed.

### 5. Verify + follow up

Claude runs the verifier (Phase 5):

- `tsc --noEmit` — types compile.
- `eslint` — no errors.
- Forbidden-pattern grep — no `any`, no hex in JSX, no module-scope Supabase clients, no hard-coded route strings (adapter mode).
- In adapter mode: the `stack/90-discipline.md` pre-submit checklist.

You get a report with:

- Files created.
- Commands you still need to run (the shadcn install, typecheck after install).
- Manual review items (copy editing, route naming, Server Action DB wiring).

---

## Vanilla mode vs. vibe-stack-rules adapter

The skill behaves differently depending on whether `stack/*.md` files are present at repo root.

| Concern | Vanilla mode | vibe-stack-rules adapter |
|---|---|---|
| `page.tsx` | Can be `async`, fetches data directly | Never `async` — static shell only |
| Data components | Inline in the page | Wrapped in `<Suspense fallback={<XSkeleton />}>` |
| Skeletons | Only if design includes them | Mandatory sibling `*.skeleton.tsx` for every data-fetching component |
| Mutations | Generated inline or via route handler if design demands | Server Action stub returning `ActionResult` in `lib/db/mutations/` |
| Route strings | Inline `href="/path"` | Entries in `lib/constants.ts` → `ROUTES.*` |
| Auth-gated pages | Not assumed | Placed under `app/(app)/` with `dynamic = "force-dynamic"` |
| Zustand | Not used unless design demands | Store Factory + Provider pattern per `stack/30-client.md` |

The adapter is automatic — no flag to pass. You opt in by having `stack/*.md` in your repo; you opt out by not having it (or by saying *"skip the vibe-stack-rules layer"* in your prompt).

---

## What the skill does **not** do

- **Run `shadcn add`** on your behalf. It emits the command; you run it.
- **Infer business logic.** Pricing, copy, and feature lists come from the design file. Dollar amounts and product names land in a `PRICING` / `FEATURES` constant in `lib/constants.ts`, not from an API you don't have.
- **Wire real database queries.** Server Action stubs and query functions are emitted with `// TODO: wire DB` markers. You fill in the Supabase calls.
- **Edit existing pages.** v1 writes new files only. If `app/(marketing)/landing/page.tsx` already exists, the plan flags it and asks whether to overwrite or pick a different segment.
- **Install npm dependencies** beyond what `shadcn add` handles. If a design implies `framer-motion` or a chart library, Claude will ask before adding it.
- **Convert non-Claude-Design inputs.** Figma, Sketch, or hand-pasted HTML won't trigger the skill. The frontmatter description specifically matches the Claude Design command shape.

---

## Troubleshooting

### The skill didn't trigger when I pasted the command

Claude Code matches skills by the frontmatter description. Make sure:

- You pasted **both lines** — the URL *and* `Implement: <file>`.
- Your URL contains `api.anthropic.com/v1/design/`.
- The `Implement:` directive is on its own line.

If it still doesn't trigger, invoke it explicitly: "Use the `claude-design-to-nextjs` skill to implement this design" followed by the command.

### `Cannot find module '@/components/ui/<x>'`

You haven't run the shadcn install yet. Run the exact command the plan emitted.

### The generated `page.tsx` is `async` but I'm using vibe-stack-rules

Check that your `stack/*.md` files are at repo root (or a parent of your current working directory). Run:

```bash
ls stack/00-foundations.md
```

If it's not there, the adapter didn't activate. Re-run the `degit` install.

### The plan missed a repeating pattern

The Component Extractor uses a 3+ repeat threshold (per `stack/30-client.md`). If the pattern only repeats twice, it stays inline — by design. If you want to force an extraction, say in the approval step: *"also extract the testimonial card, there are 2 of them but I want a reusable version."*

### The plan included a Server Action I don't want

Tell Claude during the approval checkpoint: *"drop the newsletter form and just render the section statically."* Claude updates the plan before writing files.

### `tsc` or `eslint` fails after generation

Read the verifier report — it points at the failing file and line. Common causes: missing `shadcn add`, a prop type mismatch, or a stub query function's return type not matching what the component expects. Fix in place; do not bypass the check by adding `any` or `@ts-ignore`.

---

## Example session

Your message:

```
Fetch this design file, read its readme, and implement the relevant aspects of the design.
https://api.anthropic.com/v1/design/h/abc123/?open_file=pricing.html
Implement: pricing.html
```

Claude's response flow:

1. Announces adapter detection.
2. Fetches the bundle (`WebFetch`).
3. Dispatches 3 parallel subagents (Style Mapper, Component Extractor, shadcn Matcher).
4. Synthesizes the plan, shows it to you.
5. *You approve.*
6. Writes files in dependency order (shared components → page components → skeletons → `page.tsx`).
7. Reminds you to run `npx shadcn@latest add ...`.
8. Runs `tsc --noEmit` + `eslint` + forbidden-pattern grep.
9. Emits the verification report.

Total wall time: typically 2-4 minutes for a landing-page-sized design. Most of the time is the parallel analysis + your review of the plan.

---

## Feedback

The skill lives in [template/skills/claude-design-to-nextjs/](template/skills/claude-design-to-nextjs/). If a rule is actively blocking you, open an issue with:

1. The Claude Design URL (or a minimal reproducer).
2. What the skill emitted.
3. What you expected instead.

Rule changes need to earn their keep — every addition to the reference files costs context in every future run.
