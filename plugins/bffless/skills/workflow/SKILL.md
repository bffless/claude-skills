---
name: workflow
description: Author, update, and deploy an implementation of the Workflow harness with the @bffless/workflow CLI and the publish-workflow action
---

# Workflow

**Docs**: `apps/workflow/docs/writing-an-implementation.md` and
`apps/workflow/docs/spec/06-discovery-publishing-files.md` in `bffless/apps`.

Workflow is a **harness**: one generic app (`/workflow`, alias `workflow`) that reads a YAML
script and runs it in the browser. Everything that makes a workflow *do* something specific —
its steps, its pipeline endpoints, its islands — lives in an **implementation**, a separate
package/repo that publishes into the harness's project. One harness, many implementations.

## When to use this

Author, update, or deploy an implementation of the Workflow harness — starting a new one,
adding a workflow to an existing one, or publishing a change. Not for editing the harness
itself (`apps/workflow` in `bffless/apps`), and not for a project's regular proxy rules (see
the **proxy-rules** / **rules-as-code** skills for that).

## The identity file

Every implementation carries `.bffless/workflow.json` — the discovery contract every tool in
this skill reads or writes:

```json
{ "alias": "myimpl", "harness": "workflow" }
```

- `alias` — the implementation's name: `^[a-z][a-z0-9-]*$`, unique in the project, not one of
  the reserved words `workflow` / `w` / `auth` / `_bffless`. It names the deploy alias, the
  rule set, the API prefix `/api/<alias>/…`, and the files prefix `/w/<alias>/…`.
- `harness` — the harness alias the implementation's rule set also attaches to (`workflow`
  unless the install is non-standard).

Aliases are **per-project identity**, not per-repo: deploying the same implementation package
to a second BFFless instance/project needs no rename — same alias, different `--project`.

## The six verbs

All live-verified 2026-09-01, `@bffless/workflow` CLI v1.0.0 (`npx @bffless/workflow`).
Exit codes: **0** clean/success, **1** lint errors/warnings, **2** usage/config/IO error.
Every write is preflighted — a fatal precondition is checked before anything touches disk.

### `init` — start a new implementation from any source repo

```
workflow init <alias> --from <owner>/<repo> [--path <dir>] [--ref <ref>] [--dest <dir>] --project <owner/project> [--harness-alias workflow] [--skip-existing] [--dry-run]
```

Portable: `--from` clones any readable repo (default `bffless/workflow-implementations`,
shallow), or reads a local directory in place with no clone. Inside the source,
`.bffless/workflow.json` is the discovery contract — `--path` names the package explicitly,
or the conventional `workflows/hello` default is tried, or the whole tree is searched.

The found package is staged in a disposable temp dir and run through the **boundary-aware
rename engine** there — old alias → `<alias>` — before anything is copied into `--dest`:
hyphen/underscore derivatives (`<old>-pr-1`, `<old>_jobs`) rewrite because the match has a
word boundary on both sides (`othello` is left alone), and `<old>_*.schema.yaml` files rename
too — schema filenames are identity, not incidental. Staging first means the rename pass only
ever sees files this command actually copies; it never walks (or rewrites) anything already
sitting in `--dest`.

`--project <owner/project>` is **required** — it's the BFFless project this implementation
deploys to, which is often *not* the same as the GitHub repo the package lives in — because
`init` also generates `.github/workflows/deploy-<alias>.yml` / `preview-<alias>.yml` (skipped
only when a repo-root package copies into a repo-root destination, whose own top-level CI
travels with it). Existing hand-edited workflow files at those paths are never clobbered;
they're reported as skipped.

```bash
npx @bffless/workflow init myimpl --from bffless/workflow-implementations --path workflows/hello --project my-org/my-project
```

`--dest .` copies the whole package into the current directory — fine for a fresh,
implementation-only repo, but into an **existing app** it dumps a full implementation package
(rules, workflows, its own CI) alongside that app's own files. Prefer a subdirectory `--dest`,
or a fresh repo, when the host already has its own thing going on — an implementation like
`workflow-studio` is a full application in its own right, not a snippet.

Other flags: `--skip-existing` (keep the host's version on a path collision instead of
refusing; collisions are reported under a "skipped — merge by hand" section) and `--dry-run`
(print the copy/rename/generate plan, write nothing). Without `--skip-existing`, any collision
refuses the whole command up front (exit 2, every colliding path listed).

### `rename` — re-identify an implementation in place

```
workflow rename <old> <new> [--dry-run]
```

Run from inside an already-`init`ed implementation directory. `<old>` must match what
`.bffless/workflow.json` actually declares, or the command refuses rather than guess which
tree was meant. Same rename engine as `init`'s staging pass, applied in place: the
`.bffless/proxy-rules/<old>/` directory, the identity file, schema file names/refs, and every
non-binary, non-vendored file's text content wherever `<old>` appears with a word boundary.

### `add` — scaffold a new workflow + rule stub

```
workflow add <name> [--step <path>]…
```

Run from inside an already-`init`ed implementation directory — the alias (and therefore the
rule-set directory to scaffold into) is read from `.bffless/workflow.json`. Writes
`.bffless/workflows/<name>.workflow.yaml` (one job, one `uses: pipeline` step per `--step`,
defaulting to a single step named `<name>` when `--step` is omitted) plus a matching rule
stub (`rule.yaml` + `.fn.js` + `.fn.test.yaml`) per step path.

This is the YAML↔rule contract: a step's `with.path` is relative and prefix-free
(`path: echo` → `POST /api/<alias>/echo` at publish time), and it only resolves at run time if
a rule exists at `rules/<path>/<method>/rule.yaml`. `workflow add` writes both halves together
precisely so `workflow lint` reports zero `rule-missing` findings immediately — no
hand-authoring required before the first lint pass.

### `lint` / `index`

```
workflow lint  <file...> [--json] [--quiet] [--rules <dir>] [--alias <alias>] [--path-prefix <p>]
workflow index <workflows-dir> --out <dir> --impl <alias> --name <display> [options]
```

Delegates straight into `@bffless/workflow-lint`'s own `lintFile`/`buildIndex` — same flags,
same exit-code contract as that package's `workflow` CLI. This is the same check `publish` and
`publish-workflow` run before deploying, so a failing lint is never published.

### `publish` — index, prepare, sync, deploy, attach

```
workflow publish --api-url <url> --project <owner/project> [--alias <alias>] [--harness-alias workflow] [--path <dir>] [--workflows <dir>] [--rules <dir>] [--dry-run]
```

Run from inside an already-`init`ed implementation directory. Drives the same four moves
`bffless/publish-workflow`'s GitHub Action makes, in process, against a live BFFless instance:

1. **index** — `buildIndex` builds `<path>/.bffless/workflows/index.json` from `--workflows`,
   checked against `--rules`.
2. **prepare** — an alias-named copy of the rule set is staged under a disposable temp dir,
   plus a generated `/w/<alias>/*` forwarder rule (`forwardCookies: true`, `order: 5`)
   pointing at the alias served in-process by the CE backend — never written into the source
   tree.
3. **rules push** — spawns `npx --yes bffless rules push` against the staged copy, syncing it
   under `/api/<alias>/` on `--project`.
4. **upload + attach** — zips `--path` (default `dist`) and deploys it to `<alias>` (its own
   rule set attached by name), then unions the synced rule set's id into `--harness-alias`'s
   own `proxyRuleSetIds` — idempotent, so publishing the same implementation twice is a no-op.

The API key comes from `BFFLESS_API_KEY` env **only** — never a flag, so it never lands in the
process list; a missing key exits 2 before any network call. `--dry-run` prints all four moves
with fully resolved values (URLs, alias, rule-set names, paths) and performs none of them.
Works from any checkout: commit SHA is read from git when available, a placeholder otherwise.

```bash
BFFLESS_API_KEY=… npx @bffless/workflow publish --api-url https://admin.example.com --project my-org/my-project
```

## Publish vs CI

**CI: `bffless/publish-workflow@v1`** is authoritative for real deployments — it's what
`deploy-<alias>.yml` / `preview-<alias>.yml` (generated by `init`) call, and it's also the
only thing that owns **preview teardown** (`mode: teardown` on PR close deletes the preview
alias and its rule set — the CLI has no teardown verb).

**`workflow publish` is the local/manual path** — useful for testing a publish before wiring
CI, or the first-time bring-up of a new harness install before any pipeline exists to publish
from.

**Multiple targets** (e.g. staging + production instances) are parameterized runs of the
*same* action job — different `api-url`/`api-key`/`repository` per target — never a copied
implementation. The workflow YAML and rule set stay identical between environments because
paths are always relative.

## Links

- `apps/workflow/docs/writing-an-implementation.md` (`bffless/apps`) — the full authoring walk
- `apps/workflow/docs/spec/06-discovery-publishing-files.md` (`bffless/apps`) — the discovery/
  publish contract this skill summarizes
- [`bffless/workflow-implementations`](https://github.com/bffless/workflow-implementations) —
  reference implementations: `hello` (smallest shape of every piece) and `workflow-studio`
  (full-size worked example: nine jobs, thirteen rules, two schemas, scripts, an island, a
  matrix job, headless declarations)
- `packages/workflow-cli/README.md` (`bffless/apps`) — full flag reference

## Reading a run

Runs live in the harness UI at `/workflow`; an implementation's own pages serve under
`/w/<alias>/…` on the same origin. Async job state (e.g. a long-running step polled to
completion) typically lands in a `<alias>_jobs`-style data table, the same pattern Studio uses
for its own async work — check there before assuming a stuck step has no record of what it
did. A failing `workflow lint` is never published, so a run backed by stale endpoints usually
traces back to a rule that was never authored (`rule-missing`) rather than a runtime bug.
