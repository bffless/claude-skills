---
name: upload-artifact
description: GitHub Action for uploading build artifacts to BFFless
---

# BFFless Upload Artifact Action

The `bffless/upload-artifact` GitHub Action uploads build artifacts from your CI/CD pipeline to a BFFless instance. It's available on the [GitHub Marketplace](https://github.com/bffless/upload-artifact).

For full documentation, see the [README on GitHub](https://github.com/bffless/upload-artifact).

## Quick Start

Only 3 inputs are required:

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
```

## Common Workflows

### PR Preview with Comment

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # Required for PR comments
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for commit timestamp

      - name: Build
        run: npm run build

      - uses: bffless/upload-artifact@v1
        with:
          path: dist
          api-url: ${{ vars.ASSET_HOST_URL }}
          api-key: ${{ secrets.ASSET_HOST_KEY }}
          alias: preview
          description: 'PR #${{ github.event.pull_request.number }} preview'
          pr-comment: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Production Deploy

```yaml
- uses: bffless/upload-artifact@v1
  id: deploy
  with:
    path: dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
    tags: ${{ needs.release.outputs.version }}
    description: 'Release ${{ needs.release.outputs.version }}'

- run: echo "Deployed to ${{ steps.deploy.outputs.sha-url }}"
```

### Multiple Artifacts (Same Workflow)

You can upload multiple artifacts in the same workflow:

```yaml
- name: Upload frontend
  uses: bffless/upload-artifact@v1
  with:
    path: apps/frontend/dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
    base-path: /dist

- name: Upload skills
  uses: bffless/upload-artifact@v1
  with:
    path: .bffless
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
```

## All Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `path` | **yes** | -- | Build directory to upload |
| `api-url` | **yes** | -- | BFFless platform URL |
| `api-key` | **yes** | -- | API key for authentication |
| `repository` | no | `github.repository` | Repository in `owner/repo` format |
| `commit-sha` | no | auto | Git commit SHA |
| `branch` | no | auto | Branch name |
| `is-public` | no | `'true'` | Public visibility |
| `alias` | no | -- | Deployment alias (e.g., `production`) |
| `base-path` | no | `/<path>` | URL prefix the alias serves under. See [Base path and URL shape](#base-path-and-url-shape). |
| `committed-at` | no | auto | ISO 8601 commit timestamp |
| `description` | no | -- | Human-readable description |
| `proxy-rule-set-name` | no | -- | Single proxy rule set name (legacy — prefer `proxy-rule-set-names`) |
| `proxy-rule-set-id` | no | -- | Single proxy rule set ID (legacy — prefer `proxy-rule-set-ids`) |
| `proxy-rule-set-names` | no | -- | Comma-separated proxy rule set names. Appended idempotently — re-deploying with the same names is a no-op. See [Attaching multiple proxy rule sets](#attaching-multiple-proxy-rule-sets). |
| `proxy-rule-set-ids` | no | -- | Comma-separated proxy rule set IDs. Same append-and-dedupe semantics as `proxy-rule-set-names`. |
| `tags` | no | -- | Comma-separated tags |
| `summary` | no | `'true'` | Write GitHub Step Summary |
| `summary-title` | no | `'Deployment Summary'` | Summary heading |
| `working-directory` | no | `'.'` | Working directory |
| `pr-comment` | no | `'false'` | Post comment on PR |
| `comment-header` | no | `'🚀 BFFLESS Deployment'` | PR comment header |
| `github-token` | no | `GITHUB_TOKEN` | Token for PR comments |

## Outputs

| Output | Description |
|--------|-------------|
| `deployment-url` | Primary URL (SHA-based) |
| `sha-url` | Immutable SHA-based URL |
| `alias-url` | Alias-based URL |
| `preview-url` | Preview URL (if basePath provided) |
| `branch-url` | Branch-based URL |
| `deployment-id` | API deployment ID |
| `file-count` | Number of files uploaded |
| `total-size` | Total bytes |
| `response` | Raw JSON response |

## How It Works

1. **Validates** the build directory exists
2. **Zips** the directory preserving structure
3. **Uploads** via multipart POST to `/api/deployments/zip`
4. **Sets outputs** from API response
5. **Writes Step Summary** with deployment table
6. **Posts PR comment** (if enabled)
7. **Cleans up** temporary files

## Auto-Detection

The action automatically detects:

- **Repository**: from `github.repository`
- **Commit SHA**: PR head SHA or push SHA
- **Branch**: PR head ref or push ref
- **Committed At**: via `git log` (requires `fetch-depth: 0`)
- **Base Path**: derived from `path` input as `/<path>` — chosen so files appear at the auto-alias root. See [Base path and URL shape](#base-path-and-url-shape) for what this means and when to override.

## Base path and URL shape

`base-path` controls the URL prefix the deployment's alias serves files under. It does **not** rewrite the zip — the action always zips the contents of `path` with the source folder name preserved (e.g. `path: coverage` produces a zip containing `coverage/index.html`). At serve time, the alias's `basePath` is prepended to the incoming request URL before the lookup against stored asset keys.

Combined with the default `base-path: /<path>`, this produces a useful sleight of hand: the prefix on the URL cancels the folder name on the stored path, so files appear at the **auto-alias root**.

Example: `path: coverage`, default `base-path`

| Request URL | `basePath` prepended | Resolves to stored | Result |
|---|---|---|---|
| `/index.html` | `coverage/index.html` | `coverage/index.html` | ✅ served |
| `/coverage/index.html` | `coverage/coverage/index.html` | — | ❌ 404 |

If you'd rather the source folder be **visible** in the URL (e.g. `/coverage/index.html`), set `base-path: /` so the alias's prefix is empty:

| Request URL | `basePath` prepended | Resolves to stored | Result |
|---|---|---|---|
| `/coverage/index.html` | `coverage/index.html` | `coverage/index.html` | ✅ served |
| `/index.html` | `index.html` | — | ❌ 404 |

Rule of thumb:

- **Want files at auto-alias root** (`<auto-alias>/file.png`) → leave `base-path` unset (default).
- **Want folder visible in URL** (`<auto-alias>/srcfolder/file.png`) → set `base-path: /`.
- **Want a different sub-path** → set `base-path: /custom-prefix`. The serving lookup will prepend `custom-prefix/` to incoming requests, so this only works if the zip contents are under a folder of the same name (i.e. `path: custom-prefix`).

### Pitfalls

- **Do not use `base-path: ./`** — the backend normalization only strips leading/trailing slashes, so `./` becomes the literal segment `.` and gets prepended to every lookup, breaking all requests. Use `/` instead.
- Empty / whitespace values are also unsafe; pass exactly `/` to mean "no prefix."

## Attaching multiple proxy rule sets

Use `proxy-rule-set-names` (or `proxy-rule-set-ids`) when the deployment's auto-preview alias needs to chain more than one proxy rule set — for example a Stripe webhook rule set followed by an AI proxy rule set:

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    proxy-rule-set-names: stripe-webhook,ai-proxy
```

Semantics:

- **Append + idempotent.** Each name/id is appended to whatever the alias already has. Re-running the workflow with the same list is a no-op — no duplicate join rows, no rule reordering.
- **Order preserved on first attach.** The list order is the priority order in the rule merge — earlier entries win when two rule sets match the same request.
- **Names resolve per-project.** Unknown names fail the deploy with a `400`; existing IDs that don't belong to the project are silently ignored at the rule-merge layer.
- **Singular still works.** `proxy-rule-set-name` / `proxy-rule-set-id` remain supported as a back-compat shim. If both singular and plural are provided, the plural list wins and the singular value is ignored.

If you need full replacement (drop all existing rule sets and set exactly this list), do not use the action — use the `update_alias` API with the `proxyRuleSetIds` array instead. The action's deploy path is intentionally append-only so that rule sets added through other channels (admin UI, scripts) are never silently dropped by a workflow run.

## PR Comments

When `pr-comment: true`, the action posts a formatted comment:

> ## 🚀 BFFLESS Deployment
>
> **Alias:** `preview`
>
> | Property    | Value                              |
> | ----------- | ---------------------------------- |
> | **Preview** | [example.com](https://example.com) |
> | **Commit**  | `abc1234`                          |
> | **Files**   | 42                                 |
> | **Size**    | 1.2 MB                             |

Comments are automatically updated on subsequent pushes (no duplicates).

## Troubleshooting

### Missing commit timestamp

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0 # Full history needed for git log
```

### PR comment permission denied

```yaml
permissions:
  pull-requests: write # Required for comments
```

### Files 404 at the auto-alias root

If `https://<auto-alias>/<file>` returns 404 but `https://<auto-alias>/<srcfolder>/<file>` works (or vice versa), `base-path` is the wrong shape. See [Base path and URL shape](#base-path-and-url-shape).

Common cause: setting `base-path: ./` instead of `base-path: /`. The backend doesn't normalize `./`, so it gets prepended literally and breaks every lookup.

### Custom base path

If your app expects to be served from a subdirectory (and the zip contents live under that directory):

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: build
    base-path: /docs/build # Served at /docs/build/*
```

See [Base path and URL shape](#base-path-and-url-shape) for the full mental model.
