# Quality bar

What separates a good visual plan from markup soup. Adapted from agent-native's document-quality
and wireframe standards.

## Document substance

- **Technical, not marketing.** Outcome-first prose, specific details, concrete steps. No "make it
  work", no hero headings.
- **Self-contained.** Reads on its own with zero chat context. Fold decisions into prose; never
  reference "the previous version" or "as we discussed".
- **Concrete before abstract.** Lead with one real product example (a `<Wireframe>` or `<Diff>`)
  before architecture tables. Preserve the user's intended abstraction level.
- **Right altitude.** Separate the reusable core from app-specific adapters. Show the shape of
  load-bearing files, not every line.

## Document spine

1. Objective & done-criteria (`<Meta>` + a sentence)
2. Scope / non-goals
3. Approach + key decisions (`<Callout kind="decision">`)
4. Ordered steps naming real files (`<Steps>`)
5. Risks (`<Callout kind="warn">`)
6. Verification — realistic end-to-end tests (`<Checklist>`)
7. Open questions, only if decisions remain (`<QuestionForm>`, at the very bottom)

## Don't duplicate visuals and prose

UI-heavy work: let `<Wireframe>` carry the design, the prose carry the mechanics. Backend/architecture
work inverts it: prose is the surface, with `<Diagram>` + `<Diff>` as evidence.

## Wireframe rules

- Compose from **bare flex/grid elements + `.wf-*` helper classes + `<Icon>`**. Root containers get
  padding (14–16px), `box-sizing: border-box`.
- **Tokens only** — never hard-code hex colors or set `font-family`. No decorative shadows unless the
  real product has them.
- Use `min-width: 0` and safe overflow on flex rows; avoid negative margins and absolute positioning.
- Keep labels short; `white-space: nowrap` on rows that shouldn't wrap.
- Use `mobile` surface only for genuinely phone-specific work — don't force desktop/mobile pairs.

## Anti-patterns

- Hard-coded colors/fonts/fixed pixel dimensions in MDX.
- Placeholder bars or marketing hero sections.
- Architecture diagrams as a vague left-to-right chain with overlapping labels.
- `<QuestionForm>` anywhere but the bottom.
- Hand-rolled HTML where a block exists. Reach for `custom` markup only inside `<Wireframe>` /
  `<Diagram>` children.

## Pre-handoff check

Run `npm run dev` and confirm: every block rendered, no overlap or clipped content, readable
contrast in both light and dark (the page follows `prefers-color-scheme`), and real file/symbol names
throughout.
