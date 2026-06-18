# Deploy to BFFless (optional)

Previewing locally (`npm run dev`) is the primary loop — you never need to deploy to see a plan.
Deploy only when the user wants a **shareable link**.

The gallery is plain static HTML, so deploying = building `dist/` and uploading it with the
[`bffless/upload-artifact`](https://github.com/bffless/upload-artifact) GitHub Action.

## One-time setup

In the repo that holds `visual-plans/`, set:

- **Variable** `ASSET_HOST_URL` — your instance, e.g. `https://your-instance.bffless.app`
- **Secret** `ASSET_HOST_KEY` — a BFFless API key

Then copy the shipped workflow to the **repo root** (`.github/workflows/` — GitHub only runs
workflows from there, not from subdirectories):

```bash
mkdir -p .github/workflows
cp visual-plans/.github/workflows/deploy-visual-plans.yml .github/workflows/
```

Adjust `working-directory:` if your gallery isn't at `./visual-plans`.

## What it does

On push to `main` (or manual dispatch) it runs `npm ci && npm run build` in `visual-plans/`, then:

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: visual-plans/dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: visual-plans            # stable URL hosting the whole gallery
    pr-comment: true               # on PRs, comments the preview URL
```

The action returns a stable **alias URL** (`alias-url`) and an immutable **sha URL**. With
`pr-comment: true` and `pull-requests: write` permission, PRs get the preview link as a comment.

## Manual / CI-free deploy

To deploy from a laptop without the workflow, build and POST the zip yourself — see the
`upload-artifact` skill for the raw API. The simplest path is still the action via
`workflow_dispatch`.
