---
name: rules-as-code
description: Manage proxy rule sets as code in git with the bffless CLI and CI sync
---

# Rules as Code

**Docs**: https://docs.bffless.app/recipes/proxy-rules-as-code/

The `bffless` CLI (npm package `bffless`) authors a proxy rule set as **files in git**
instead of only through the admin UI/MCP: one YAML manifest per route, handler bodies as
real `.fn.js`/`.fn.ts` files you can lint and test, and a compiler that turns the directory
back into the export JSON the dashboard's Import already accepts. Requires CE ≥ 0.2.0 for
the live sync endpoints (`push`/`diff`/`pull`/revisions).

## When to use this (vs dashboard/MCP edits)

Use rules-as-code for a rule set that should be code-reviewed, tested, and deployed with the
app it backs — CI syncs it on every merge. Keep using the dashboard/MCP (see the
**proxy-rules** skill) for ad-hoc exploration or a set nobody owns in a repo. Both share the
same server-side data — a set is never locked into one mode; a dashboard edit to a
git-managed set is warned, not blocked (see [Troubleshooting](#troubleshooting)).

## Directory layout

```
.bffless/
  config.json                    # { apiUrl?, project?, ruleSets? } — no secrets, committable
  proxy-rules/
    <set-name>/
      ruleset.yaml                # { name, description?, environment? }
      schemas/items.schema.yaml   # pipeline schemas, referenced by name (never UUID)
      rules/
        api/feeds/remove/post/    # directory form — this rule has code files
          rule.yaml
          pick.fn.js
          pick.fn.test.yaml
        api/items/get.rule.yaml   # single-file form — no code, no directory needed
        api/items/[...path]/any.rule.yaml
      dist/                       # written by `rules build`; gitignored
```

A rule's path is derived Next.js-router-style from its directory under `rules/` (trailing
`[...x]/` → `*`; `pathPattern:` is the escape hatch for anything else). `method`/`methods`
comes from the filename stem (`get.rule.yaml`, `any.rule.yaml`); `pipeline:` is authoring
sugar over the canonical `pipelineConfig` shape.

## CLI commands

| Command | Purpose | Key flags |
|---|---|---|
| `rules init [dir]` | Scaffold authoring files (currently: a schema manifest) | `--schema <name>`, `--field <name:type[:required]>` (repeatable), `--force` |
| `rules build [dirs...]` | Compile authoring dir(s) to export JSON | `-o <file>` (single dir) |
| `rules validate [dirs...]` | Lint manifests + handler code, no network | — |
| `rules test [dirs...]` | Run `*.fn.test.yaml` fixtures in a `node:vm` harness | — |
| `rules pull [set-name]` | Fetch a live set and decompile to authoring layout | `--from-file`, `--decompile`, `-o <dir>`, `--force` |
| `rules diff [dirs...]` | Compare local vs. live (CI drift check) | exit `0` sync / `1` drift / `2` error |
| `rules push [dirs...]` | Compile + idempotently sync to the server | `--dry-run`, `--prune`, `--strict-schemas`, `--name-suffix <suffix>` |
| `rules dev [dirs...]` | Local-first watch: rebuild/validate/test on save | `--push` (requires `--name-suffix`), `--api-url`/`--api-key`/`--project` |
| `rules revisions <set>` | List a set's revision history (last 20 kept) | `--api-url`/`--api-key`/`--project` |
| `rules rollback <set>` | Revert a set to a prior revision (a real sync) | `--to <revisionId>` (default: newest non-current), `--dry-run`, `--api-url`/`--api-key`/`--project` |

`[dirs...]` defaults to the nearest `.bffless/config.json`'s `ruleSets` glob array when
omitted. `pull`/`push`/`diff`/`dev`/`revisions`/`rollback` all accept `--api-url`, `--api-key`,
`--project` overrides.

## Schemas: the name is the identity

There is **no chicken-and-egg step** for pipeline schemas — never pre-create one in the
dashboard to obtain an id. Author `schemas/<name>.schema.yaml` as `{ name, fields }` (no
`id`), reference it from rules as `$schema:<name>`, and `rules push` resolves by name:
an existing same-name project schema is reused, a missing one is **created by the sync**.
Scaffold one with:

```bash
bffless rules init --schema comments --field author:string:required --field body:text
```

Field types: `string | number | boolean | email | text | datetime | json`; the trailing
modifier is `required`/`optional` (default optional). One sharp edge: **push never changes
the fields of an existing live schema** — the live definition wins (mismatch = warning, or
a hard error with `push --strict-schemas`). Settle fields before the first push; after
that, live field changes happen in the dashboard. Also note `--name-suffix` (PR previews)
suffixes only the *rule set* name — schemas are project-level, so preview sets share the
same named schemas and data tables as production.

## Config & auth

`.bffless/config.json` (`{ apiUrl?, project?, ruleSets? }`) is discovered walking up from
`cwd` and is **safe to commit** — it never holds secrets.

- **API URL**: `--api-url` > `BFFLESS_API_URL` env > config `apiUrl`.
- **API key**: `--api-key` > `BFFLESS_API_KEY` env **only** — never read from config. Never
  commit an API key; set it as a CI secret, not a repo variable.
- **Project**: `--project` > config `project` (UUID, `owner/name`, or bare name).

## Authoring handlers

A `.fn.js` `code:` ref is inlined **byte-verbatim** as the compiled
`function_handler.config.code` — write real, indented, multi-line source, never a minified
one-liner; see the **pipelines** skill ("Authoring Handler Code via MCP") for the full
formatting rule and sandbox constraints (no `crypto`/`Buffer`/`require`/`process`/`fetch`).

A `.ts` ref is **bundled with esbuild** instead of raw-read:

- Entry must `export default function handler(ctx) {...}` (or `export function handler`).
- Imports are **relative only, confined to the rule set directory** — a bare specifier or an
  escaping path is a build error. No `node_modules` resolution.
- `import type { HandlerContext, ... } from 'bffless/handlers'` is fine (type-only, erased).
- **No typechecking at build time** — esbuild transpiles syntax only; run `tsc --noEmit`
  yourself for real type safety.
- The bundled output runs through the same prohibited-pattern lint as a `.fn.js` file, so an
  unsafe *imported* util fails the build even if the raw `.fn.ts` looks clean.
- `rules pull --decompile` never regenerates TypeScript — only `.fn.js` — and warns (doesn't
  block) if the target already has hand-authored `*.fn.ts` files.

## Testing (`*.fn.test.yaml`)

```yaml
handler: ./pick.fn.js
cases:
  - name: returns first query row
    data: { steps: { query: [{ id: 1, url: "https://a" }] } }
    expect: { result: { id: 1, url: "https://a" } }
  - name: throws when query is missing
    data: { steps: {} }
    expect: { throws: "Cannot read" }
```

`handler` is relative to the fixture; `expect` is exactly one of `result` (deep-equality) or
`throws` (substring match). Fixtures are discovered anywhere under `rules/`; a bad one only
fails its own case(s).

## CI recipe

Real apps sync with `bffless/deploy-proxy-rules@v1` (wraps `rules push`), then deploy with
`bffless/upload-artifact@v1` attaching the same set by name — **sync before upload**, so the
build's proxy rules exist before it goes live:

```yaml
- uses: bffless/deploy-proxy-rules@v1
  with:
    path: apps/reader/.bffless/proxy-rules/reader
    api-url: ${{ vars.BFFLESS_URL }}
    api-key: ${{ secrets.BFFLESS_API_KEY }}
    project: ${{ vars.BFFLESS_PROJECT }}

- uses: bffless/upload-artifact@v1
  with:
    path: apps/reader/dist
    api-url: ${{ vars.BFFLESS_URL }}
    api-key: ${{ secrets.BFFLESS_API_KEY }}
    alias: reader
    proxy-rule-set-names: reader
```

**PR previews**: on `pull_request`, pass `name-suffix: pr-<N>` so the sync targets
`<set-name>-pr-<N>` — a fresh live set, never production — then attach it to a per-PR alias.
A `cleanup-*` workflow on PR close deletes the alias first (a rule set 409s while attached),
then the suffixed rule set.

**Drift check**: a scheduled workflow runs `bffless rules diff` and fails on exit `1` so
manual dashboard edits to a git-managed set get noticed.

## Revisions & rollback

The server keeps the **last 20 revisions** of a set, captured after every sync, import, or
dashboard edit (best-effort, deduped on identical content). `rules revisions <set>` lists
them newest-first, each flagged `current` against the live state. `rules rollback <set>
[--to <revisionId>] [--dry-run]` replays a snapshot as a real sync (default target: newest
non-current) — it never renames/recreates the set, and a non-dry-run rollback itself
captures a new revision (history only moves forward, like `git revert`). A rule deleted then
restored by rollback loses any secret header values it carried — the report warns. Prefer
`git revert` when the change is already in git; reach for `rules rollback` when drift
happened outside git (a dashboard edit, a bad manual sync) and you want the server fixed now.

## Troubleshooting

**`rules diff` reports unexpected drift?**
- Someone edited the set in the dashboard — `rules pull <set-name>` before re-authoring, or
  `rules push` to overwrite it. A reused schema name with mismatched fields only warns
  without `--strict-schemas`; with it, it's a hard error.

**Dashboard shows "Managed from git — manual edits will be overwritten"?**
- Expected: edits are allowed but not sticky. Run `rules pull` to capture the change before
  the next `rules push` discards it.

**`rules push`/`rules rollback` fails with a 400 about multiple rules on one path pattern?**
- A **methods-split** set (two rules on the same path, each with a distinct `methods:` list
  and no single `method`) is legal live state but ambiguous to sync/replay. Consolidate to
  one `any.rule.yaml` with a combined `methods:` list, or give each rule an explicit `method:`.

**Leftover PR-preview rule sets/aliases?**
- Check for `<set>-pr-<N>` sets after a PR is force-closed or cleanup was disabled mid-PR;
  delete the alias before the rule set (it 409s while attached).
