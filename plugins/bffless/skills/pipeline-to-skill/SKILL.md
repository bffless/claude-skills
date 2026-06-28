---
name: pipeline-to-skill
description: Generate a project-scoped agent skill from a BFFless proxy rule set so an agent can call the app's dynamic API
---

# Pipeline to Skill

Turn a BFFless app's proxy rule set (its `/api/*` pipelines) into a portable `<app>-api/SKILL.md`
that teaches an agent how to call it. The output is a standard `SKILL.md` (name/description
frontmatter) — the cross-agent format the `skills` CLI installs — so it works in any runtime, not
just Claude Code. The output lives in the *app's* repo; this generator is platform knowledge.

## Inputs

1. **Rule set** — prefer the committed export `<app>.proxy-rules.json` in the app repo; else
   fetch live with `get_proxy_rule_set` via the BFFless MCP.
2. **App repo context** — the API client (e.g. `src/store/*Api.ts`) and any `CONTEXT.md`
   glossary, for accurate names, request bodies, and multi-step flows.

## Steps

1. **Enumerate endpoints** — for each rule read `method` + `pathPattern`.
2. **Recover params** — scrape `request.body.*` / `request.query.*` references from each rule's
   `function_handler`/handler config to list parameter names. When an API client is provided as
   context, prefer its declared types (e.g. the `RegisterBody`/`PreparedUpload` interfaces in a
   `*Api.ts`/`nodes.ts`) over inferred-unknown — the client carries the precise request/response
   shapes the rules only hint at.
3. **Match handler patterns → recipes:**
   - `presigned_upload` (+ `register_upload`): emit the upload recipe — `prepare` → `PUT <uploadUrl>`
     (direct to bucket, no key) → register node. Cross-reference the app client to confirm the
     register body, since this flow is not visible in any single rule.
   - `data_create` / `data_query`: CRUD recipe using the referenced schema's fields.
   - share-link / signed-url / auth-relay handlers: document their known request/response flow.
   - generic `function_handler` + `response_handler`: best-effort request/response shape, clearly
     marked as inferred.
4. **Write the standard auth section** — the *pattern* is identical for every BFFless app, but the
   concrete values are environment-specific. **Discover them; never hardcode `j5s-dev` or `j5s.dev`**
   (those are just the reference platform's MCP-entry name and domain):
   - **Credential** — a BFFless API key for the project, sent as `X-API-Key`. Source it
     runtime-neutrally; **do not assume the agent is Claude Code**:
     1. Primary: the `BFFLESS_API_KEY` environment variable (`-H "X-API-Key: $BFFLESS_API_KEY"`) —
        works in any agent runtime (Claude Code, Copilot, Gemini CLI, Codex, …).
     2. Convenience: if a BFFless MCP is already configured, reuse its key so nothing new is stored.
        Read it from that runtime's MCP config — on Claude Code, `~/.claude.json`, selecting the
        `mcpServers` entry **by its `url`** (the admin MCP endpoint `https://admin.<platform-domain>/mcp`),
        since the entry *name* is user-chosen. Other runtimes keep MCP config elsewhere — give the
        Claude path as one labelled example, not the only path.
     Derive `<platform-domain>` from the admin MCP url's host (or the app's deployment).
   - **App base URL** — the alias the rule set is attached to on that platform, e.g.
     `https://<app-alias>.<platform-domain>`.
   - **Bake the concrete discovered values** into the generated skill: lead with the env-var path
     (`$BFFLESS_API_KEY`); for the MCP-reuse convenience, emit a `python3` one-liner that selects the
     entry **by `url`** (matching the admin MCP url), not by name; and make requests target the app's
     real base URL. Do not emit the literal `j5s-dev`/`j5s.dev` unless it is genuinely this
     environment's value.
   - Identity = project owner. The skill itself stores nothing and never writes the key value into a
     file (the env var / MCP config is the operator's, not the skill's).
   - Note the gate: an `X-API-Key` is accepted on rules that allow it (e.g. an `allowApiKey: true`
     validator); rules without it fall back to cookie/session auth. Flag any endpoint a key cannot reach.
5. **Assemble** sections: front-matter (`name: <app>-api`), intro, Auth, Discovery, one recipe
   per significant endpoint, Gotchas (note private-by-default ACL and presigned/no-key steps).
6. **Write** the `SKILL.md` into the app repo at the location the consuming agent loads project
   skills from — `.claude/skills/<app>-api/SKILL.md` for Claude Code; other runtimes (Copilot,
   Gemini CLI, Codex) use their own skills location, or install via the `skills` CLI. The file
   contents are identical across runtimes; only the path differs.

## Self-check

Read back the generated skill and confirm: every endpoint in the rule set is represented or
intentionally omitted; multi-step flows (uploads) are whole recipes, not single calls; the auth
section is present and stores nothing new. If the API client references endpoints absent from the
rule set (or vice versa), document the gap explicitly rather than silently dropping it.
