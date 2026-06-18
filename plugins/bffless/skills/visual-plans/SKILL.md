---
name: visual-plans
description: Turn an implementation plan into a scannable visual document — built from reusable React blocks (wireframes, schemas, API contracts, diffs, diagrams), authored as MDX, previewed locally, and optionally deployed to BFFless. Use when the user wants a "visual plan", a shareable plan page, or a richer alternative to a plain markdown plan before writing code.
---

# Visual Plans

Produce a **visual plan**: a self-contained Astro + React + MDX page built from a fixed catalog of
reusable block components, so every plan looks consistent and is easy to scan. The plan renders to
static HTML you preview **locally** (`npm run dev`) — deploying to BFFless is optional.

This is the self-hosted counterpart to a SaaS planning tool: no external service, no account. The
artifact is HTML in your repo.

> Companion skill: **visual-recaps** turns a *finished diff/PR* into a review summary using the same
> components. Use this skill for forward planning, that one for "what changed".

## When to use

Use for modest UI changes, multi-file work, ambiguous scope, or any change the user wants to **see,
compare, or approve before code**. Skip it for one-line fixes describable in a sentence.

## Workflow

### 1. Scaffold once (if needed)

If a `./visual-plans/` gallery does not already exist in the working directory, create it by copying
this skill's `template/` directory, then install:

```bash
cp -R "<this-skill-dir>/template" ./visual-plans
cd ./visual-plans && npm install
```

`<this-skill-dir>` is the directory containing this SKILL.md. If the gallery already exists, skip
straight to step 3 — every plan is just a new file in it.

### 2. Research first (read-only)

Inspect the actual codebase before authoring. Name **real** files, symbols, routes, and data shapes
— never invent them. Check existing patterns and reuse them before proposing new ones. No source
edits during planning; the plan is the approval gate.

### 3. Author one MDX page

Create `visual-plans/src/content/plans/<YYYY-MM-DD>-<slug>.mdx`. Start with frontmatter and a
`<Meta>` header, then compose blocks. Map content to the right block — do not hand-roll markup:

| Content | Block |
| --- | --- |
| The load-bearing choice | `<Callout kind="decision">` |
| Schema / data shape | `<DataModel>` with `change` flags |
| HTTP endpoint | `<ApiEndpoint>` |
| UI screen | `<Wireframe>` (flex/grid + `.wf-*` helpers + `<Icon>`) |
| Before/after, option A/B | `<Columns>` |
| Architecture / data flow | `<Diagram>` |
| Files touched | `<FileTree>` |
| A specific code change | `<Diff>` |
| Ordered work | `<Steps>` (name files per step) |
| Acceptance / verification | `<Checklist>` |
| Remaining decisions (bottom only) | `<QuestionForm>` |

Follow the document spine: **objective & done-criteria → scope/non-goals → approach + key decisions
→ ordered steps (real files) → risks → verification → open questions**. See
`references/components.md` for every block's props and `references/quality.md` for the quality bar.
The shipped `template/src/content/plans/example-plan.mdx` shows all blocks in use.

### 4. Preview locally

```bash
cd visual-plans && npm run dev    # http://localhost:4321
```

Hand the user the local URL. Verify there is no overlap, clipping, or unreadable contrast, and that
every block rendered. This is the deliverable — **no deploy required to see it**.

### 5. Deploy (optional)

Only when the user wants a shareable link. See `references/deploy.md`.

## Rules

- **Reuse the blocks** — consistency is the whole point. If something doesn't fit a block, prefer a
  small composition of `<Wireframe>`/`<Diagram>` children over bespoke HTML.
- **Tokens only** — components are styled by `--wf-*` tokens (`src/styles/tokens.css`). Never
  hard-code hex colors or font families in MDX.
- **Concrete before abstract** — lead with a real product example, then the architecture.
- **Self-contained** — the page must read on its own, with no reference to chat history.
- **Import from `@components`** — e.g. `import { Meta, Callout, DataModel } from '@components';`.
