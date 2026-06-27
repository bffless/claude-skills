---
name: pipeline-to-skill
description: Generate a project-scoped Claude Code skill from a BFFless proxy rule set so an agent can call the app's dynamic API
---

# Pipeline to Skill

Turn a BFFless app's proxy rule set (its `/api/*` pipelines) into a `.claude/skills/<app>-api/SKILL.md`
that teaches an agent how to call it. The output lives in the *app's* repo; this generator is
platform knowledge.

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
4. **Write the standard auth section** (identical for every BFFless app):
   reuse the j5s-dev MCP `X-API-Key` from `~/.claude.json`
   (`mcpServers.j5s-dev.headers.X-API-Key`); send as `X-API-Key`; identity = project owner;
   base URL = the app's attached alias. Never store a new credential or write the key into the file.
   Note the gate: an `X-API-Key` is accepted on rules that allow it (e.g. an `allowApiKey: true`
   validator); rules without it fall back to cookie/session auth. Flag any endpoint a key cannot
   reach so an agent does not assume the key works everywhere.
5. **Assemble** sections: front-matter (`name: <app>-api`), intro, Auth, Discovery, one recipe
   per significant endpoint, Gotchas (note private-by-default ACL and presigned/no-key steps).
6. **Write** to `<app-repo>/.claude/skills/<app>-api/SKILL.md`.

## Self-check

Read back the generated skill and confirm: every endpoint in the rule set is represented or
intentionally omitted; multi-step flows (uploads) are whole recipes, not single calls; the auth
section is present and stores nothing new. If the API client references endpoints absent from the
rule set (or vice versa), document the gap explicitly rather than silently dropping it.
