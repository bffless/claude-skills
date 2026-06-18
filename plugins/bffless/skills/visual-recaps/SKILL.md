---
name: visual-recaps
description: Turn a finished diff or PR into a scannable visual review summary — file tree, annotated diffs, schema/API deltas, and diagrams — authored as MDX, previewed locally, and optionally deployed to BFFless. Use when the user wants a "visual recap", a review summary of what changed, or a shareable PR walkthrough. The inverse of visual-plans.
---

# Visual Recaps

Produce a **visual recap**: a self-contained page that maps an actual diff/PR to reusable block
components, so a reviewer can grasp what changed at a glance. Same engine as **visual-plans** —
forward planning is that skill; "what shipped" is this one. Both share one gallery and one component
catalog.

## When to use

After implementing a change, or to summarize a PR/branch for review. Recap the **whole work unit**,
separating the changes this work introduced from any pre-existing unrelated edits. Skip for trivial
one-line changes.

## Workflow

### 1. Scaffold once (if needed)

The recap lives in the **same `./visual-plans/` gallery** as plans. If it doesn't exist yet, copy
the template from the **visual-plans** skill and install:

```bash
cp -R "<visual-plans-skill-dir>/template" ./visual-plans
cd ./visual-plans && npm install
```

If the gallery already exists, skip to step 3.

### 2. Read the real diff

Base the recap on actual changed lines — `git diff`, the PR, the files. Extract **real** paths,
fields, methods, and line stats. Never infer facts. **Redact secrets** (keys, tokens, signed URLs)
as `<redacted>` / `sk-•••`.

### 3. Author one MDX page

Create `visual-plans/src/content/recaps/<YYYY-MM-DD>-<slug>.mdx`. Frontmatter uses `summary` and an
optional `pr` field. Map the diff mechanically to blocks:

| Change | Block |
| --- | --- |
| Overview of files touched + stats | `<FileTree>` with `change` flags |
| Schema change | `<DataModel>` with `change: 'added'\|'removed'\|'changed'` |
| New/changed endpoint | `<ApiEndpoint change="added">` |
| A key code hunk | `<Diff>` (one-line summary + the lines) |
| Before/after behavior | `<Columns>` |
| Architecture shift | `<Diagram>` |
| UI change | `<Wireframe>` (before/after) |
| Verified outcome | `<Callout kind="ok">` |

Keep it lean but substantial: combine a structural summary (`<FileTree>`, `<DataModel>`) with
implementation evidence (annotated `<Diff>`). Aim for ≤ ~8 key-change sections. The shipped
`template/src/content/recaps/example-recap.mdx` is a worked example. Block props are in the
visual-plans skill's `references/components.md`.

### 4. Preview locally

```bash
cd visual-plans && npm run dev    # http://localhost:4321
```

Hand the user the local URL. **No deploy required to see it.**

### 5. Deploy (optional)

Shareable link only — see the visual-plans skill's `references/deploy.md`.

## Rules

- Map to facts in the diff; one-line summary per `<Diff>`, then the real lines.
- Reuse blocks — never hand-roll markup. Tokens only; no hard-coded colors/fonts.
- Redact secrets. Recap the whole thread, flagging unrelated pre-existing edits separately.
- Import from `@components`: `import { Meta, FileTree, DataModel, Diff } from '@components';`.
