# vibe-stack-rules

> Modular, import-friendly engineering rules for vibe coders shipping on **Next.js + Supabase + shadcn/ui + Zustand**.
>
> Drop them at the root of your repo. Your agent behaves.

If you've watched Claude Code, Cursor, Codex, or any other vibe-coding agent ship the same 10 bugs on this stack over and over — async `page.tsx`, module-scope Supabase clients, Zustand state leaks, ISR on authenticated routes, duplicate zod schemas, skeletons buried in `if (!data)` branches — this repo is the fix.

---

## Why this is split into multiple files

A monolithic 500-line `CLAUDE.md` is a trap. Three problems:

1. **Context pollution.** Your agent loads every rule on every session, even when editing a button label.
2. **Adherence degrades with length.** Anthropic's own docs recommend keeping memory files under ~200 lines for reliable rule-following. Longer = more drift.
3. **Wrong home for generic rules.** `CLAUDE.md` should be the brain of your specific app — what it does, how to run it, current decisions — not a stack tutorial.

The fix: a **lean `CLAUDE.md`** at your repo root that describes your project, plus a **`stack/` folder** of focused rule files that `CLAUDE.md` imports via `@path` syntax. Each stack file is ~150–250 lines, scoped to one concern, and loaded by Claude Code automatically.

---

## This repo's structure

```
vibe-stack-rules/
├── README.md                    ← you are here
├── LICENSE                      ← MIT
├── .gitignore
└── template/                    ← the files you install into your project
    ├── CLAUDE.md                ← lean project-brain template
    ├── stack/
    │   ├── 00-foundations.md    ← reading instructions, versions, principles, project structure
    │   ├── 10-server.md         ← RSC boundary, data fetching, Suspense, Server Actions
    │   ├── 20-supabase.md       ← 3 clients, middleware/proxy, auth gate, the 6 traps
    │   ├── 30-client.md         ← Zustand store factory + shadcn composition
    │   └── 90-discipline.md     ← redundancy rules, TS rules, LLM checklist, anti-patterns
    └── .claude/
        └── skills/
            └── claude-design-to-nextjs/   ← convert Claude Design bundles → Next.js + TSX + Tailwind + shadcn
                ├── SKILL.md
                └── references/            ← per-phase subagent prompts + stack adapter
```

Everything inside `template/` is what lands in your project. The rest is repo metadata.

## What lands in your project after install

```
your-next-app/
├── CLAUDE.md                    ← customize this with your project details
├── stack/                       ← leave these alone unless amending rules
│   ├── 00-foundations.md
│   ├── 10-server.md
│   ├── 20-supabase.md
│   ├── 30-client.md
│   └── 90-discipline.md
├── .claude/
│   └── skills/
│       └── claude-design-to-nextjs/  ← triggers when you paste a Claude Design command
├── app/                         ← your actual Next.js app
├── lib/
└── ...
```

`CLAUDE.md` at your repo root imports the `stack/*.md` files via `@stack/*.md` syntax. Claude Code picks them up automatically at session start.

---

## The target stack

| Tool | Version | Role |
|---|---|---|
| Next.js | ≥ 15 (App Router) | Framework |
| React | ≥ 19 | RSC + Server Actions |
| TypeScript | strict mode | Type safety |
| Supabase | `@supabase/ssr` ≥ 0.10 | Auth + database |
| shadcn/ui | latest | Component primitives |
| Tailwind CSS | ≥ 3.4 | Styling |
| Zustand | ≥ 5 | Client UI state |

> **Middleware filename:** Next.js ≤15 uses `middleware.ts` at project root. Next.js 16+ renamed it to `proxy.ts`. Same code, different filename. The rules cover both.

---

## Install

### 1. Copy the template into your project

The cleanest way is `degit`, which pulls the `template/` folder contents directly into your current directory without any git history:

```bash
# from your project root
npx degit imfaisii/vibe-stack-rules/template .
```

That single command creates `CLAUDE.md` and `stack/` at the root of your project. Done.

**Alternative** (no Node required):

```bash
# from your project root
curl -L https://github.com/imfaisii/vibe-stack-rules/archive/main.tar.gz | tar -xz
cp -r vibe-stack-rules-main/template/. .
rm -rf vibe-stack-rules-main
```

**Agent-specific filenames:**
- **Claude Code** → reads `CLAUDE.md` automatically
- **Cursor** → also reads `CLAUDE.md`, or rename to `.cursorrules`
- **Codex / newer agents** → reads `AGENTS.md` (the emerging cross-agent standard)
- **Next.js** ships with native `AGENTS.md` support in recent versions, so `AGENTS.md` is the most future-proof filename

If you run multiple agents, symlink the extras:

```bash
ln -s CLAUDE.md AGENTS.md
```

### 2. Install the Supabase agent skills

Supabase ships a set of agent skills that keep your coding agent up to date on the latest SSR patterns, RLS, edge functions, and schema generation — without you needing to paste docs into context every time.

```bash
npx skills add supabase/agent-skills
```

This is the single most important thing you can do after dropping the rules files. Without it, your agent will keep reaching for the deprecated `@supabase/auth-helpers-*` packages from its training data.

### 3. Customize `CLAUDE.md` for your project

Open `CLAUDE.md` at your repo root and fill in:

- Project name + one-liner
- Your actual package manager commands (`npm` / `pnpm` / `yarn` / `bun`)
- Architecture notes specific to your app
- Known gotchas that keep biting you

**⚠️ Do not delete the `@stack/*.md` import lines** at the bottom of `CLAUDE.md`. Those are what pull the rule files into Claude Code's context. The file itself has a prominent warning about this.

### 4. (Optional) Generate your database types

The rules require a single source of truth for database types at `src/types/database.ts`:

```bash
npx supabase gen types typescript --project-id <your-project-id> > src/types/database.ts
```

Add it as a `db:types` script in `package.json` and re-run after every schema change. Never hand-edit this file.

---

## How Claude Code loads the rules

When you open a session in a project with this setup, Claude Code:

1. Reads `CLAUDE.md` at the repo root.
2. Sees the `@stack/00-foundations.md`, `@stack/10-server.md`, etc. import lines.
3. Automatically expands each imported file into context alongside `CLAUDE.md`.
4. Applies them as persistent instructions for the session.

The `@path` import syntax is recursive up to 5 levels deep, so you can further split individual stack files if they grow too long — or add your own.

You can verify what's loaded at any time inside Claude Code with:

```
/memory
```

---

## What the rules enforce (the short version)

- **Server Components by default.** `"use client"` is an escape hatch.
- **`page.tsx` is never `async`.** Static shell renders instantly; data streams in via `<Suspense>` + skeletons.
- **Every data-fetching component ships with a sibling `*.skeleton.tsx`.**
- **Supabase clients are created per-request, never at module scope.**
- **`getUser()`, not `getSession()`**, for server-side authorization.
- **`export const dynamic = "force-dynamic"` is required** on any route reading session cookies. No ISR on auth routes, ever.
- **Zustand uses the Store Factory + Provider pattern.** Per-request stores. No global module-scope `create()`.
- **Zustand is client-only, UI-only.** Server data belongs to RSC.
- **Single source of truth** for every type, schema, constant, and route. Duplicates are treated as bugs.
- **All mutations go through Server Actions** returning a standard `ActionResult` shape.
- **All shadcn usage goes through `cn()`** for className composition.
- **Domain wrappers > raw shadcn trees.** Repeat the same `<Card><CardHeader>…</Card>` three times? Extract it.

Full details in `template/stack/`.

---

## Bundled skill — Claude Design → Next.js

Anthropic launched **Claude Design** on 2026-04-17. It spits out HTML + CSS + component bundles behind a URL, along with a standardized command:

```
Fetch this design file, read its readme, and implement the relevant aspects of the design.
https://api.anthropic.com/v1/design/h/<id>?open_file=<file>
Implement: <file>
```

The raw output isn't TSX, isn't Tailwind, and definitely isn't following any of the rules in `stack/*.md`. `template/.claude/skills/claude-design-to-nextjs/` closes that gap.

**What it does:**

- Triggers automatically when you paste the Claude Design command.
- Runs a 5-phase pipeline: **Ingest → Analyze (3 subagents in parallel) → Synthesize + approval checkpoint → Generate → Verify**.
- Converts raw HTML/CSS to Next.js App Router code with TypeScript, Tailwind tokens, and shadcn/ui primitives.
- Extracts reusable components when a pattern repeats 3+ times in the bundle (following `stack/30-client.md`).
- **Detects this repo's stack rules at run time.** When `stack/*.md` is present, the skill additionally enforces non-async `page.tsx`, sibling `*.skeleton.tsx` files, Suspense boundaries, `ROUTES` constants, Server Action stubs returning `ActionResult`, and `cn()` composition. In a vanilla Next.js project, it skips all of that and stays framework-clean.

**What it does not do:** run `shadcn add` on your behalf (emits the command; you run it), infer business logic (stubs are marked `// TODO`), or edit existing pages (writes new files only).

### Installing just the skill

Claude Code auto-discovers skills from two locations:

- **Project-local:** `.claude/skills/<skill-name>/` — shared with anyone who clones the repo.
- **User-global:** `~/.claude/skills/<skill-name>/` — available in every project on your machine.

Pick whichever scope fits. All three options below work without any manual linking — Claude Code picks them up at session start.

**Option 1 — with the full vibe-stack-rules bundle (recommended)**

Gets you the stack rules *and* the skill, so the adapter auto-activates:

```bash
npx degit imfaisii/vibe-stack-rules/template .
```

**Option 2 — skill only, project-scoped**

Skips the stack rules; the skill will run in vanilla mode:

```bash
npx degit imfaisii/vibe-stack-rules/template/.claude/skills/claude-design-to-nextjs .claude/skills/claude-design-to-nextjs
```

**Option 3 — skill only, user-global**

Installs for every project on your machine:

```bash
npx degit imfaisii/vibe-stack-rules/template/.claude/skills/claude-design-to-nextjs ~/.claude/skills/claude-design-to-nextjs
```

Verify it loaded inside Claude Code with `/memory` — the skill's frontmatter shows up under the available-skills list.

### Quick start

1. Open [claude.ai/design](https://claude.ai/design) and describe the UI you want. Copy the "Fetch this design file..." command it hands you.
2. Paste the command into Claude Code inside your project. The skill auto-triggers.
3. Review the conversion plan Claude shows you (shadcn install list, files to be created, extracted components) and approve or request edits.
4. Run the `npx shadcn@latest add ...` command the skill emits.
5. Claude writes the files, runs `tsc` + `eslint`, and reports any follow-up.

Full walkthrough with prerequisites, vanilla-vs-adapter behavior, and troubleshooting: **[USAGE.md](USAGE.md)**.

---


---

## Who this is for

- Solo founders shipping fast with AI coding agents
- Teams using Cursor / Claude Code / Codex as their primary dev interface
- Anyone tired of reviewing PRs where the agent re-invented a type that already exists three folders over
- Anyone who's been bitten by a Supabase session leaking between users on Vercel

It is **not** a gentle introduction to Next.js. It assumes you already know the stack and you're optimizing for the agent that's writing the code, not the human that's reading it for the first time.

---

## Using it with your agent

1. Drop `CLAUDE.md` + `stack/` into your repo root (via `degit`, step 1 above).
2. Tell your agent, once, at the start of a session: "Read CLAUDE.md and the imported stack files before touching any code. Treat them as law."
3. When the agent proposes a diff, ask it to run the pre-submit checklist in `stack/90-discipline.md` against its own output.
4. When the agent tries to break a rule, it should surface the conflict instead of silently routing around it. If it doesn't, reinforce the "When these files are wrong" section at the bottom of `stack/00-foundations.md`.

---

## Contributing

This is an opinionated file set. Opinions can be wrong. If you've found a rule that's actively blocking good code, open an issue with:

1. Which file + which rule
2. The concrete scenario it blocks
3. A proposed amendment

PRs that add new rules need to justify why the rule is worth the cognitive load. Every addition has to earn its keep — the whole point of splitting the files was to stay lean.

---

## License

MIT. Use it, fork it, remix it, ship it. See `LICENSE`.

---

## Credits

Distilled from a lot of production pain on the Next.js + Supabase stack. If it saves you one late-night incident, we're even.
