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
1. INGEST         (1 agent, sequential)
2. ANALYZE        (3 subagents in parallel — Task tool with subagent_type=general-purpose)
3. SYNTHESIZE     (1 agent, waits for user approval of the plan)
4. GENERATE       (1 agent, sequential — shared components first, then routes)
5. VERIFY         (1 agent — tsc + eslint + pre-submit checklist)
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

**Do not write any files yet.** Pass the inventory into Phase 2.

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
4. **Never skip the Phase 3 approval checkpoint.** The user must see the plan before files are written.
5. **Never mix adapter rules into vanilla mode.** If `adapter = "none"`, do not add `_components/` folders, Suspense boundaries, `ROUTES`, or Server Action stubs unless the source design explicitly demands them.
6. **Never write imports that resolve to nothing.** Dependency order in Phase 4 is strict.

---

## Reference files (load per phase)

- `references/analyze-agents.md` — the 3 parallel subagent prompts
- `references/synthesize-and-generate.md` — plan assembly + file-writing rules
- `references/verify.md` — verification checklist
- `references/adapter-vibe-stack-rules.md` — conditional stack-specific layering
