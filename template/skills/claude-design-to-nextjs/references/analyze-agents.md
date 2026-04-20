# Analyze phase — 3 parallel subagent prompts

Dispatch these **together in a single message with three `Agent` tool calls** so they run concurrently. Each subagent gets the Phase 1 inventory (HTML target, referenced files, CSS, README) and the relevant prompt below.

> **Orchestrator rule:** Do not merge results until all three return. Then hand the combined result to Phase 3 (synthesizer).

---

## Subagent 1 — Style Mapper

**subagent_type:** `general-purpose`
**description:** `CSS → Tailwind + tokens mapping`

**Prompt template:**

```
You are the Style Mapper subagent in the claude-design-to-nextjs pipeline.

You receive a Claude Design bundle (HTML + CSS + README). Your job is to produce a
style migration plan. You DO NOT write code. You return a structured report.

INPUTS
- All CSS content from the bundle (global + scoped)
- The README (may define design tokens)
- The target HTML file(s)

OUTPUTS — return exactly this JSON shape

{
  "tailwindConfig": {
    "tokens": {
      "colors": { "<token-name>": "<value>" },
      "fontFamily": { "<name>": ["<stack>"] },
      "fontSize": { "<name>": "<value>" },
      "spacing": { "<name>": "<value>" },
      "borderRadius": { "<name>": "<value>" },
      "boxShadow": { "<name>": "<value>" }
    },
    "plugins": ["<plugin-name>", ...]
  },
  "cssVariables": {
    ":root": { "<var-name>": "<value>", ... },
    ".dark": { "<var-name>": "<value>", ... }
  },
  "utilityMap": [
    { "selector": ".btn-primary", "tailwind": "inline-flex items-center gap-2 rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary/90" },
    ...
  ],
  "darkModeStrategy": "class" | "media" | "none",
  "warnings": [
    "<anything that can't be mapped cleanly — e.g. animations with @keyframes, container queries, etc.>"
  ]
}

RULES
- Use shadcn/ui's canonical token names when possible: primary, primary-foreground,
  secondary, muted, destructive, border, ring, background, foreground, card,
  popover, accent. Map the design's tokens to these names, don't invent new ones.
- Prefer Tailwind spacing scale (0, 0.5, 1, 1.5, 2 ... 96) over arbitrary px values.
  Only emit arbitrary values (p-[13px]) when absolutely necessary, and flag them.
- For any CSS that can't be expressed as Tailwind utilities (keyframes, complex
  pseudos), put it in cssVariables or globals.css additions and warn.
- Do NOT rewrite the HTML. You only plan the style layer.
```

---

## Subagent 2 — Component Extractor

**subagent_type:** `general-purpose`
**description:** `Detect repeating patterns, propose reusable components`

**Prompt template:**

```
You are the Component Extractor subagent in the claude-design-to-nextjs pipeline.

You receive the target HTML file plus any referenced partials. Your job is to
find repeating structural patterns and propose reusable components.

THRESHOLD
- A pattern qualifies for extraction if it repeats 3+ times in the bundle.
- If it only repeats twice, keep it inline. Do NOT over-extract.

OUTPUTS — return exactly this JSON shape

{
  "extractions": [
    {
      "name": "StatCard",                           // PascalCase, domain-meaningful
      "rationale": "<one sentence: what it is, why extract>",
      "occurrences": 4,                             // count in the bundle
      "proposedPath": "components/common/stat-card.tsx",
      "props": [
        { "name": "title", "type": "string", "required": true },
        { "name": "value", "type": "string | number", "required": true },
        { "name": "delta", "type": "string", "required": false },
        { "name": "className", "type": "string", "required": false }
      ],
      "shadcnUsed": ["Card", "CardHeader", "CardContent"],
      "exampleSnippet": "<first occurrence's HTML, trimmed>"
    }
  ],
  "inlineKeep": [
    "<one-line description of any 2x pattern deliberately kept inline>"
  ],
  "warnings": [
    "<anything ambiguous — e.g. similar but not identical patterns that may or may not be the same component>"
  ]
}

RULES
- Naming: prefer domain-meaningful names (StatCard, PricingTile, FeatureRow)
  over structural ones (ThreeColumnCard, BigBoxWithIcon).
- Every extraction must accept an optional `className` prop for composition.
- Every extraction must be pure presentational — no data fetching, no state.
  (State/data wiring is the caller's job.)
- Prefer prop-driven over slot-driven unless the content truly varies
  in structure (then use children or named slots).
- If two patterns are almost-but-not-quite the same, either (a) unify them
  with a variant prop, or (b) extract them separately and warn. Never merge
  two patterns that happen to look similar but mean different things.
```

---

## Subagent 3 — shadcn Matcher

**subagent_type:** `general-purpose`
**description:** `Map HTML elements to shadcn primitives, emit install list`

**Prompt template:**

```
You are the shadcn Matcher subagent in the claude-design-to-nextjs pipeline.

You receive the target HTML file plus any referenced partials. Your job is to
map raw HTML/CSS patterns to shadcn/ui primitives and emit the exact CLI
install command the user will need to run.

OUTPUTS — return exactly this JSON shape

{
  "installCommand": "npx shadcn@latest add button card dialog input form label skeleton",
  "primitiveMap": [
    {
      "sourceSelector": "button.btn-primary",
      "primitive": "Button",
      "variant": "default",
      "size": "default",
      "notes": "Maps directly. Apply cn() for any utility overrides."
    },
    {
      "sourceSelector": ".card",
      "primitive": "Card",
      "subComponents": ["CardHeader", "CardTitle", "CardDescription", "CardContent", "CardFooter"],
      "notes": "Wrap in a domain component (e.g. StatCard) rather than repeating the <Card>...<CardHeader>... tree inline."
    }
  ],
  "gaps": [
    "<any HTML patterns with no clean shadcn match — e.g. a custom rating widget>"
  ],
  "warnings": [
    "<anything that could be a shadcn primitive but looks deliberately diverged in the design>"
  ]
}

RULES — canonical mappings (use these names exactly)

| HTML pattern                              | shadcn primitive            |
|-------------------------------------------|-----------------------------|
| <button>                                  | Button                      |
| <a class="btn-*">                         | Button asChild with <Link>  |
| .card / .panel                            | Card + sub-components       |
| <input type="text|email|password|...">    | Input                       |
| <textarea>                                | Textarea                    |
| <select>                                  | Select + SelectTrigger etc. |
| <input type="checkbox">                   | Checkbox                    |
| <input type="radio">                      | RadioGroup + RadioGroupItem |
| <input type="range">                      | Slider                      |
| <input type="file">                       | Input type="file"           |
| modal / dialog / overlay                  | Dialog                      |
| side panel / drawer                       | Sheet                       |
| tooltip                                   | Tooltip                     |
| dropdown menu                             | DropdownMenu                |
| tabs                                      | Tabs                        |
| accordion / collapse                      | Accordion                   |
| toast / notification                      | Sonner (toast)              |
| avatar                                    | Avatar                      |
| badge / pill                              | Badge                       |
| breadcrumb                                | Breadcrumb                  |
| progress bar                              | Progress                    |
| table                                     | Table                       |
| skeleton / loader                         | Skeleton                    |
| separator / divider                       | Separator                   |
| scroll area                               | ScrollArea                  |
| form (with validation)                    | Form + react-hook-form + zod|
| label                                     | Label                       |

- DO NOT invent shadcn components. If a pattern isn't in the canonical map,
  list it under `gaps` with a suggested fallback (plain Tailwind JSX or
  a custom component).
- ALWAYS include `skeleton` in the install command if the adapter is on or
  if any card/list/table/chart appears in the design.
- The install command must be ONE line that the user can copy-paste.
```

---

## Merging the three outputs

The synthesizer (Phase 3) receives all three JSON payloads and:

1. Deduplicates shadcn primitives across the matcher and extractor results.
2. Resolves conflicts (e.g. Style Mapper wants to name a token `brand` but shadcn Matcher expects `primary` — shadcn wins; the design's `brand` becomes an alias).
3. Sequences the extractions so components referenced by other components come first.
4. Feeds the combined plan into the user approval checkpoint.

See `synthesize-and-generate.md` for the synthesis rules.
