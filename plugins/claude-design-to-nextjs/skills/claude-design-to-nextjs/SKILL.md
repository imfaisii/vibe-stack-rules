---
name: claude-design-to-nextjs
description: Use when the user pastes a Claude Design command like "Fetch this design file, read its readme, and implement... https://api.anthropic.com/v1/design/h/<id>?open_file=<file>  Implement: <file>". Converts the fetched Claude Design bundle (HTML + CSS + components + README) into production Next.js App Router code — TypeScript + Tailwind + shadcn/ui. Extracts shared components when patterns repeat 3+ times. Automatically layers the vibe-stack-rules contract (non-async page.tsx, static-shell Suspense, sibling skeletons, RSC boundary, Server Actions, ROUTES constant, cn() composition) when it detects a stack/ folder with stack/*.md files at the repo root. Stays framework-clean (no Zustand/Supabase assumptions) in vanilla Next.js projects.
---

# Claude Design → Next.js conversion skill

> **Purpose:** Turn a Claude Design bundle into idiomatic, production-grade Next.js code that either drops straight into a vibe-stack-rules project or into a vanilla Next.js App Router project.

---

## When this skill fires

The user pastes a command shaped like one of:

```
Fetch this design file, read its readme, and implement the relevant aspects of the design.
https://api.anthropic.com/v1/design/h/<id>?open_file=<file>
Implement: <file>
```

or anything containing `api.anthropic.com/v1/design/` plus an `Implement:` directive. Treat the URL + target filename as the skill's input.

## Adapter detection — do this first

Before fetching anything, detect whether the vibe-stack-rules contract should be layered on top of the generic generator.

```
if any of these exist at repo root OR under the current working dir:
  - stack/00-foundations.md
  - stack/10-server.md
  - stack/20-supabase.md
  - stack/30-client.md
  - stack/90-discipline.md
then:
  adapter = "vibe-stack-rules"   → also load references/adapter-vibe-stack-rules.md
else:
  adapter = "none"
```

Announce the adapter choice to the user in one sentence before Phase 1 starts.

---

## The pipeline (topology B: parallel analyze, sequential generate)

```
1.   INGEST             (1 agent, sequential)
1.5. STYLE MODE         (mandatory checkpoint — ask the user every run, no defaults)
2.   ANALYZE            (3 subagents in parallel — Task tool with subagent_type=general-purpose)
3.   SYNTHESIZE         (1 agent, waits for user approval of the plan)
4.   GENERATE           (1 agent, sequential — shared components first, then routes)
5.   VERIFY             (1 agent — tsc + eslint + pre-submit checklist)
```

All heavy per-phase guidance lives in `references/`. Load each file on-demand per phase — do not preload.

### Phase 1 — Ingest

1. Parse the user's message for the design URL and the `Implement:` target filename.
2. `WebFetch` the URL.
3. Locate and read the bundle's README (Claude Design ships one; it describes the file layout and any design tokens).
4. Fetch the target file (e.g. `landing.html`) and every file it references (component HTML partials, CSS files, images).
5. Build an in-memory inventory:
   - target file path
   - referenced component files
   - CSS files (global + scoped)
   - asset list (images/SVG/fonts)
   - tokens/variables from the README or CSS `:root`

**Do not write any files yet.** Pass the inventory into Phase 1.5.

### Phase 1.5 — Style mode checkpoint (MANDATORY, ASK EVERY RUN)

**This step is not optional and not cacheable.** Ask the user this question on every single invocation, even if you think you know their answer, even if they answered the same way last time. Claude Design bundles ship static CSS files — without this explicit question, Claude tends to keep the CSS files alongside the JSX (which defeats the point of using this skill).

Present exactly this question block to the user and stop for their answer:

```markdown
## Style conversion mode

Claude Design ships static CSS. How should the generator handle styles?

**A. Full Tailwind conversion (recommended).**
   Every CSS class from the bundle is mapped to Tailwind utilities and
   applied directly in the JSX. The CSS files are NOT copied into the
   project. `globals.css` only gets CSS variables / design tokens +
   anything Tailwind genuinely can't express (rare).

**B. Hybrid.**
   Simple rules (padding, margin, color, typography) → Tailwind utilities.
   Complex rules (@keyframes, container queries, cascade layers, complex
   pseudo-selectors) stay in a CSS module co-located with the component.
   Use this if the design uses advanced CSS features.

**C. Preserve CSS files.**
   The original CSS files are copied alongside the components, class
   names are kept as-is in the JSX. No Tailwind conversion. Use this
   only when you specifically want to retain the source stylesheet
   structure.

Which mode? (A / B / C)
```

Once the user answers, record the mode in an inline variable for the remaining phases:

- **Mode A** → Style Mapper runs in full-conversion mode. Generator replaces every source className with the mapped Tailwind utilities. No CSS file imports in JSX.
- **Mode B** → Style Mapper separates utility-convertible rules from CSS-required rules. Generator applies Tailwind utilities AND emits a `.module.css` file alongside each component that needs the complex rules.
- **Mode C** → Style Mapper is skipped entirely (no Tailwind work needed). Generator copies CSS files into `styles/` or a component-adjacent location and keeps the original classNames in JSX.

**Never skip this step. Never assume mode A just because vibe-stack-rules is active.** The adapter enforces that `globals.css` contains only CSS variables, but the *source* of the classNames (Tailwind vs CSS module vs plain CSS import) is the user's call — ask them.

### Phase 2 — Analyze (parallel)

Dispatch three subagents in a **single message with three `Agent` tool calls** so they run concurrently. Each receives the Phase 1 inventory and a link to its section in `references/analyze-agents.md`.

| Subagent | Output |
|---|---|
| **Style Mapper** | CSS → Tailwind utility mapping. Design tokens → `tailwind.config.ts` extensions. Dark-mode strategy. |
| **Component Extractor** | List of candidate reusable components (patterns repeating ≥3 times). Proposed name + API for each. |
| **shadcn Matcher** | Element-to-primitive map (`<button class="btn-primary">` → shadcn `Button variant="default"`). `shadcn@latest add <list>` install command. |

Full subagent prompts: see `references/analyze-agents.md`.

### Phase 3 — Synthesize + user approval checkpoint

1. Merge the three analyses into a single **conversion plan** containing:
   - `shadcn add` command (exact list)
   - File manifest (what gets written, where, in what order)
   - Shared-component extractions (name + purpose)
   - Tailwind config changes (tokens, theme extensions, plugins)
   - Any open questions or ambiguities worth surfacing
2. **Show the plan to the user and wait for approval before generating.** This is a mandatory checkpoint — do not skip it.
3. Apply the `vibe-stack-rules` adapter layer if active (adds `.skeleton.tsx` siblings, Suspense wiring, non-async `page.tsx`, Server Action stubs, `ROUTES` entries).

Full synthesizer rules: see `references/synthesize-and-generate.md`.

### Phase 4 — Generate

Write files in dependency order. Never write a file that imports something that doesn't exist yet.

```
1. lib/utils.ts                           (if cn() missing — should already exist)
2. lib/constants.ts                       (ROUTES updates if adapter on)
3. tailwind.config.ts                     (token/theme updates)
4. app/globals.css                        (CSS variable updates only; no hex values in JSX)
5. components/ui/*                        (shadcn primitives — emit the `shadcn add` command; do NOT hand-write them)
6. components/common/*.tsx                (extracted shared components)
7. app/<route>/_components/*.tsx          (route-private components)
8. app/<route>/_components/*.skeleton.tsx (if adapter on — one per data-fetching component)
9. app/<route>/page.tsx                   (wires Suspense + imports; non-async if adapter on)
10. app/<route>/error.tsx, loading.tsx    (if referenced)
```

Full generation rules + adapter rules: see `references/synthesize-and-generate.md` and `references/adapter-vibe-stack-rules.md`.

### Phase 5 — Verify

Run the verifier checklist. If the adapter is on, also run the vibe-stack-rules pre-submit checklist from `stack/90-discipline.md`. Report the diff summary + any failures.

Do not claim "done" until the verifier passes. If a check fails, fix it and re-run — do not ship red.

Full verification steps: see `references/verify.md`.

---

## Invariants (non-negotiable)

1. **Never hand-write a shadcn primitive.** Emit the `shadcn add` command; let the user run it.
2. **Never inline hex colors in JSX.** Use Tailwind tokens or CSS variables.
3. **Never use `any`.** `unknown` + type guard, or a proper type.
4. **Never skip the Phase 1.5 style-mode checkpoint.** The question is asked on every invocation — no defaults, no inference from previous runs, no shortcutting even when vibe-stack-rules is active. The user explicitly chooses A/B/C every time.
5. **Never skip the Phase 3 approval checkpoint.** The user must see the plan before files are written.
6. **Never mix adapter rules into vanilla mode.** If `adapter = "none"`, do not add `_components/` folders, Suspense boundaries, `ROUTES`, or Server Action stubs unless the source design explicitly demands them.
7. **Never write imports that resolve to nothing.** Dependency order in Phase 4 is strict.
8. **Never keep source CSS class names when the user chose Mode A or B.** Replacing every className with the Style Mapper's utility output is the whole point of those modes. Importing the source CSS file next to the JSX is a silent failure mode — do not let it happen.

---

## Reference files (load per phase)

- `references/analyze-agents.md` — the 3 parallel subagent prompts
- `references/synthesize-and-generate.md` — plan assembly + file-writing rules
- `references/verify.md` — verification checklist
- `references/adapter-vibe-stack-rules.md` — conditional stack-specific layering
