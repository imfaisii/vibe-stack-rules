# CLAUDE.md

> **This is the project brain.** Describe this specific app here — what it does, how to run it, architectural decisions, known gotchas. Keep it short. The generic stack rules live in `stack/` and are imported below.

---

## Project

**Name:** <your-app-name>
**One-liner:** <what this app does in one sentence>
**Stack:** Next.js (App Router) · Supabase · shadcn/ui · Zustand · TypeScript

## Commands

> Replace `<pm>` with your project's package manager: `npm`, `pnpm`, `yarn`, or `bun`. Check `package.json` and any lockfile to confirm before editing.

```bash
<pm> dev          # local dev server
<pm> build        # production build
<pm> lint         # eslint
<pm> typecheck    # tsc --noEmit
<pm> db:types     # regenerate src/types/database.ts from Supabase
```

## Architecture notes

- <e.g. "Auth gate lives in `app/(app)/layout.tsx` — protected routes must be inside that group">
- <e.g. "We use Supabase Edge Functions for webhook handlers, not Next.js route handlers">
- <e.g. "Revenue aggregation runs on a daily cron — see `supabase/functions/daily-rollup/`">

## Known gotchas

- <anything that's bitten you that you don't want to re-explain every session>

---

## Stack rules (imported) — DO NOT DELETE THIS SECTION

> **⚠️ IMPORTANT — to any human or LLM editing this file:**
> The `@stack/*.md` lines below are **import directives**, not decoration. Claude Code reads them at session start and pulls the linked files into context. Deleting, renaming, or commenting them out **silently disables the entire engineering contract** for this project.
>
> When you customize this `CLAUDE.md` with project-specific details (name, commands, architecture notes, gotchas), **leave this section untouched**. Add your content above it, not in place of it. If you genuinely need to drop a stack file (e.g. the project doesn't use Zustand), delete only that one line and note why in a comment.

The following files define the engineering contract for this stack. Treat them as law. When a rule conflicts with a request, surface the conflict before breaking the rule.

- @stack/00-foundations.md
- @stack/10-server.md
- @stack/20-supabase.md
- @stack/30-client.md
- @stack/90-discipline.md

---

## Working agreement

1. Before writing code, re-read the relevant `stack/*.md` file(s) for the area you're touching.
2. Run the pre-submit checklist in `stack/90-discipline.md` against your own diff before claiming done.
3. If you're about to duplicate a type, schema, constant, or query that already exists — **stop and import it instead**.
4. If you're about to put an `await` in `page.tsx` — **stop and wrap the data component in `<Suspense>`**.
5. If you're about to declare a Supabase client at module scope — **stop and move it inside the function**.
