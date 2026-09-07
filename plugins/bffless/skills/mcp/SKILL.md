---
name: mcp
description: Build an MCP server on a BFFless project with the mcp_handler pipeline step — tools as sibling rules, scopes, OAuth discovery, and connecting claude.ai or Claude Code
---

# MCP Server (mcp_handler)

**Docs**: https://docs.bffless.dev/features/build-an-mcp-server/

A rule set with one `mcp_handler` step **is** an MCP server. Each tool is a sibling rule of
the same alias; the sibling's `auth_required` validator is the tool's gate. Nothing is
registered in CE and no server process runs: attach the set to an alias and the endpoint
answers on that alias's hosts. Push a new version of the set and `tools/list` changes with it.

**Not this one?** The server that lets an agent drive the BFFless **admin panel** (projects,
deployments, proxy rules, …) is CE's built-in admin MCP server at `https://admin.<host>/mcp`,
authenticated with an `X-API-Key` header — see https://docs.bffless.dev/features/mcp-server/
and the **pipeline-to-skill** skill. This skill is about a server **you build on your own
project**, whose tools are your pipelines.

**Version gates**: CE ≥ 0.4.44 for `mcp_handler`, ≥ 0.4.49 for the admin-UI form editor,
≥ **0.4.52** for the `oauth_protected_resource` discovery handler (before that, see
[Legacy discovery](#legacy-discovery-ce--0452)).

**Live examples**: `bffless/presentations` → `.bffless/proxy-rules/images/` (one tool,
`generate_image`); `bffless/apps` → `apps/workflow/.bffless/proxy-rules/workflow/` (tools +
`ui://` resources for an MCP App).

## When to use this

"Add an MCP server to this project", "expose these pipelines as tools", "let claude.ai call
my app", "connect Claude Code to my BFFless backend". Not for calling the admin MCP tools
(`create_proxy_rule` etc.) — those belong to the **proxy-rules** and **pipelines** skills.

## How it works

- **Stateless Streamable HTTP.** The endpoint answers one JSON-RPC message per `POST`.
  `GET`/`DELETE` → `405` (no SSE, no sessions); a notification → empty `202`; every answer
  is `Cache-Control: no-store`. Declare `methods: [GET, POST, DELETE]` so the 405 is CE's,
  not a routing miss.
- **Tools are sibling rules.** A tool declares `rule.path` and `rule.method` (`GET` or `POST`,
  default `POST`). On `tools/call` CE runs that rule **in-process as the caller** with the
  arguments as `request.body` (POST) or `request.query` (GET). The sibling's validators run
  as usual, so its `auth_required` is the gate: `roles` for *who*, `requiredScopes` for
  *what the credential was delegated*.
- **Results are `CallToolResult`s.** A sibling answering JSON with a `content` array is passed
  through verbatim. Any other `2xx` is wrapped: body → `content[0].text` + `structuredContent`
  (a string body becomes `{ text }`). `401` → `isError` with `errors.auth`; `403` for a
  missing scope → `errors.scope` naming it; any other non-2xx → `errors.pipeline: "<code>:
  <message>"` with the HTTP status in `_meta.bffless.status`.
- **`initialize`** answers `serverInfo`, `instructions` (if set) and
  `capabilities: { tools, resources }`. `protocolVersions` (newest first) defaults to the
  versions the MCP spec has published — leave it out.

## The minimum server (rules-as-code)

Three files in the set (see the **rules-as-code** skill for the layout):

```
.bffless/proxy-rules/<set>/
  ruleset.yaml
  rules/
    api/mcp/any.rule.yaml                 # the endpoint
    api/mcp-tools/generate/post/rule.yaml # one tool
    _custom/well-known/get.rule.yaml      # OAuth discovery — needed for claude.ai
```

**Endpoint** — `rules/api/mcp/any.rule.yaml`:

```yaml
methods: [GET, POST, DELETE]
targetUrl: pipeline
order: 30
pipeline:
  name: images MCP endpoint
  steps:
    - id: mcp
      name: mcp
      handler: mcp_handler
      config:
        serverInfo: { name: bffless-presentations-images, version: 0.1.0 }
        instructions: "One tool, generate_image. Each call costs money — confirm the prompt with the person first."
        tools:
          - name: generate_image
            description: "Generate one image. Returns a temporary URL — download it to keep it."
            inputSchema:
              type: object
              properties:
                prompt: { type: string, description: "Subject, style, composition, palette." }
                aspect_ratio: { type: string, enum: ["16:9", "1:1", "9:16"] }
              required: [prompt]
              additionalProperties: false
            annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: false, openWorldHint: true }
            rule: { path: /api/mcp-tools/generate, method: POST }
  validators:
    - type: auth_required
```

The endpoint's `auth_required` has **no config**: any signed-in caller may list tools, and an
anonymous call gets the `401` that starts OAuth discovery. Do not put `roles`/`requiredScopes`
here — that gates `tools/list` for everyone; gate each tool on its own sibling.

**Tool** — `rules/api/mcp-tools/generate/post/rule.yaml`. Any pipeline works; the arguments
arrive as `request.body`:

```yaml
targetUrl: pipeline
order: 40
pipeline:
  name: MCP tool generate_image
  steps:
    - id: generate
      name: generate
      handler: replicate
      config:
        model: google/nano-banana-2
        input:
          prompt: request.body.prompt
          aspect_ratio: request.body.aspect_ratio
    - id: respond
      name: respond
      handler: response_handler
      config:
        status: 200
        contentType: application/json
        headers: { Cache-Control: no-store }
        body: '{"content":[{"type":"text","text":"Image ready: {{steps.generate.output.0}}"}],"structuredContent":{"url":"{{steps.generate.output.0}}"}}'
  validators:
    - type: auth_required
      config:
        roles: [admin]
        requiredScopes: [images:generate]
```

Answer the `content` array yourself when you want to control the text the model reads
(agent hosts show the model `content[0].text` only; put data in `structuredContent`). To
refuse a call, answer `{"isError": true, "content": [{"type":"text","text":"…"}]}` with a
2xx status — a non-2xx is also turned into an error result, but loses your wording.

**Tool declaration fields** (`McpToolDecl`): `name` (unique; `a/b` in a call is matched as
`a.b`), `description`, `inputSchema` (JSON Schema object), `annotations`
(`readOnlyHint`/`destructiveHint`/`idempotentHint`/`openWorldHint`), `visibility`
(`[model]` default, `[app]` = only the embedded MCP App calls it), `_meta` (e.g.
`_meta.ui.resourceUri`), `rule: { path, method? }`.

### Admin UI / MCP admin tools

The same shape can be built in the admin UI: a rule for `/api/mcp`, methods `GET, POST,
DELETE`, target **Pipeline**, one step of type **MCP Server** (tabs: Server, Tools with a
sibling-rule picker and a property-table schema editor, Resources, JSON). The **JSON** tab is
the whole `config` — paste it into rules-as-code. `create_proxy_rule`/`update_proxy_rule`
over the admin MCP write the same `mcp_handler` step config. The validator form exposes
**Required Roles**; `requiredScopes` is rules-as-code / JSON only today.

A `bffless rules push` overwrites dashboard edits — pick one source of truth per set.

## Auth ladder

| Credential | Sent as | Scope check | Needs the discovery rule? | Use for |
| --- | --- | --- | --- | --- |
| Session cookie | browser session on the alias host | passes every check | no | a browser-embedded client on the same host |
| API key | `X-API-Key` header | passes every check | no | your own Claude Code, scripts, CI |
| OAuth app token | `Authorization: Bearer bfat_…` | **enforced** per tool (`requiredScopes`) | **yes** | claude.ai connectors; any client that should get only consented scopes |

- `requiredScopes` uses `namespace:verb` (lowercase, `SCOPE_PATTERN`). Scopes are your app's
  vocabulary; nothing is registered in CE. A session or API key is a person acting as
  themselves, not a delegation, so it is never scope-checked. An app token's effective
  permission is the member's own permissions ∩ the token's scopes — it never elevates.
- An app token is **not** an API key: `allowApiKey` is neither needed for it nor checked
  by the validator on current CE (the flag is accepted but not enforced). Leave it out.
- `roles` matches the caller's **global** role (`admin`, `user`, `member`). An API key is
  never `admin` (see the **authorization** skill).
- In a tool's pipeline the caller is `user.*`: `user.id`, `user.email`, `user.role`, and for
  tokens `user.scopes` and `user.credential` (`'app_token'`).

## OAuth discovery (RFC 9728) — one step

claude.ai (and Claude Code with `type: http` and no header) has no place for a key. It
expects the server to be an OAuth **protected resource**: on `401` it reads
`GET https://<host>/.well-known/oauth-protected-resource`, then CE's authorization-server
metadata, registers itself (RFC 7591, public client), runs PKCE authorize + consent on the
admin host, and calls back with a bearer **app token**. CE's admin host already is the
authorization server; the only thing your set must add is this rule
(`rules/_custom/well-known/get.rule.yaml`):

```yaml
pathPattern: /.well-known/oauth-protected-resource*
targetUrl: pipeline
order: 32
pipeline:
  name: OAuth protected-resource metadata
  steps:
    - id: prm
      name: prm
      handler: oauth_protected_resource
      config:
        resource: /api/mcp            # the mcp_handler rule's path on this host (literal, no wildcard)
        # scopes: [images:generate]   # optional allowlist — see below
        # resourceName: "…"           # defaults to serverInfo.name
        # resourceDocumentation: https://…
```

What the handler does for you:

- `resource` is `https://<request host><resource>`; `authorization_servers` is CE's **real
  issuer** (`OAUTH_ISSUER` → `ADMIN_DOMAIN` → `FRONTEND_URL`), never a guess.
- `bypassVisibility` is **implied** — the rule is served on a private deployment because
  the caller has no credential yet. The implication is narrow: the rule must answer the
  well-known path and this step must be its first enabled step.
- The path-suffixed form a client tries first (`…/oauth-protected-resource/api/mcp`) answers
  the same document; a suffix naming another path is `404`. Cached 5 minutes.
- Admin UI: same rule, method GET, one step of type **OAuth Discovery (MCP)** under *Other*.
  The `mcp_handler` step's **Server** tab shows where discovery comes from and whether
  `scopes_supported` is **declared** or **derived** (with the list) — check there.

**`scopes` is a security setting, not metadata.** `scopes_supported` is the allowlist the
consent page and token grant enforce: a client asking for a scope outside it gets
`invalid_scope`; a client asking for nothing is granted the whole list. Omitted, it is the
union of `requiredScopes` on the `auth_required` validators of the tools' sibling rules —
right when every tool should be reachable over OAuth. Declare it explicitly when a sibling
scope must **not** be grantable to an OAuth client (e.g. one meant for CI API keys only). A
tool whose scope is missing from the list can never be called with an OAuth token.

Probe with no credential — must be `200` even on a private deployment:

```bash
curl -s https://<host>/.well-known/oauth-protected-resource
```

### Legacy discovery (CE < 0.4.52)

Older instances have no `oauth_protected_resource` handler. There the same document is a
`function_handler` deriving every URL from the request host + a `response_handler`, on a
rule with `bypassVisibility: true`. Copy it from `bffless/presentations` →
`.bffless/proxy-rules/images/rules/_custom/well-known/` (`get.rule.yaml`, `wellKnown.fn.js`,
its test). Two cautions: `authorization_servers` is guessed as `admin.<parent domain>` — wrong
on an instance whose admin host is elsewhere, so hard-code the issuer there; and
`scopes_supported` is whatever the function lists, so keep it in step with the tools'
`requiredScopes`. Replace it with the one-step rule after upgrading (an explicit rule at that
path keeps winning, so the copy keeps working until you do).

## Connect a client

**claude.ai**: Settings → Connectors → *Add custom connector*, URL `https://<host>/api/mcp`.
It runs the OAuth flow; the person narrows scopes on the consent page; revoke later under
**User Settings → App Tokens** in the admin UI.

**Claude Code, OAuth** — commit a `.mcp.json` with no secret in it, then `/mcp` to sign in:

```json
{ "mcpServers": { "images": { "type": "http", "url": "https://<host>/api/mcp" } } }
```

**Claude Code, API key** — skip OAuth for your own use:

```bash
claude mcp add --transport http images https://<host>/api/mcp --header "X-API-Key: YOUR_API_KEY"
```

**Probe by hand** (JSON-RPC over POST; an API key or a session cookie works):

```bash
curl -s -X POST https://<host>/api/mcp -H "Content-Type: application/json" -H "X-API-Key: YOUR_API_KEY" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
curl -s -X POST https://<host>/api/mcp -H "Content-Type: application/json" -H "X-API-Key: YOUR_API_KEY" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"generate_image","arguments":{"prompt":"a hero image"}}}'
```

Expect `401` with a `WWW-Authenticate: Bearer resource_metadata="…"` hint when you send no
credential — that is the discovery entry point, not a bug.

## `ui://` resources (MCP Apps)

Optional `resources` on the same step serve HTML for a host like claude.ai to embed:

```yaml
resources:
  static:
    - uri: ui://bffless/workflow/step-view.html
      name: Workflow step view
      rule:
        path: /api/workflow/mcp-resources/step-view   # GET; answers text/html
  templates:
    - uriTemplate: "ui://bffless/{impl}/{path+}"      # RFC 6570 level 1: {var} one segment, {var+} slash-carrying tail
      name: island
      rule:
        path: "/w/{impl}/{path+}"                     # same variables on the sibling path (quote: braces are YAML flow syntax)
  list:
    rule:
      path: /api/workflow/mcp-resources               # answers the resources array (or { resources })
      method: GET
  csp:
    connectDomains: [$app, $storage]                  # $app = request origin, $storage = the storage backend's
    resourceDomains: [$storage]
```

Every listed/read resource carries `_meta.ui.csp` from `csp`; default `mimeType` is
`text/html;profile=mcp-app`. A tool names its view with `_meta.ui.resourceUri`; `visibility: [app]`
marks a tool only the embedded app calls. Resource siblings are read with `GET` and their
own validators apply (`allowApiKey`/`requiredScopes` as for tools). Reference: the Workflow
app's set and `apps/workflow/docs/writing-an-implementation.md` in `bffless/apps`.

## Gotchas

1. **The tool's gate is on the sibling, not the endpoint.** Roles/scopes on the endpoint gate
   `tools/list` for everyone; put them on each tool rule.
2. **A tool's sibling must be a pipeline rule in a set attached to the same alias.** A proxy
   (`targetUrl: https://…`) sibling fails with `unsupported rule type`; a sibling that is
   itself an `mcp_handler` fails with `MCP_RECURSION`; no match → `<tool> is declared but no
   rule answers <path>`. The admin UI's "answered by" hint is advisory (another attached set
   may answer), so a push is never blocked — probe `tools/call`.
3. **`rule.method` is GET or POST only.** Arguments go as query (GET) or body (POST); a tool
   rule at `…/post/rule.yaml` answers POST — match them.
4. **Version gates are per instance.** Before assuming `oauth_protected_resource`, check the
   instance's CE version (admin UI); below 0.4.52 ship the legacy copy.
5. **`scopes` omitted = every sibling scope is grantable over OAuth.** Declare it to narrow.
6. **The legacy well-known copy guesses `admin.<parent domain>`** as the authorization server.
   Only the new handler reads the real issuer.
7. **`allowApiKey` does nothing for app tokens** and is not enforced on current CE; do not
   reach for it to "enable OAuth".
8. **An app token is bound to one project**: a token minted for another project's host
   answers `token_project_mismatch` (403). Tools on a scoped alias also need a project role.
9. **One document per MCP endpoint.** CE resolves discovery by matching the step whose
   `resource` equals the MCP path — two MCP servers on one alias need two well-known rules
   (the path-suffixed form disambiguates).
10. **`tools/call` arguments are not validated against `inputSchema` by CE** — validate in the
    tool rule (a `function_handler` first step that answers `{ ok: false, error }` and skips
    the expensive step via `condition`, as the images set does).

## Troubleshooting

- **claude.ai cannot connect / never shows consent** — discovery rule missing, or its
  `resource` ≠ the endpoint path. `curl` the well-known URL with no credential: must be 200.
  On a legacy copy also check `bypassVisibility: true` and `authorization_servers`.
- **`invalid_scope` at authorize** — the client asked for a scope outside `scopes_supported`.
  Add it to `scopes`, or (derived) to a tool rule's `requiredScopes`.
- **`insufficient_scope: missing …`** on a call — the token was consented with fewer scopes.
  Reconnect and grant it. Sessions/API keys never hit this.
- **`needs a signed-in caller`** — the sibling answered 401: the endpoint let an anonymous
  caller through (missing `auth_required`) or the call was cross-project.
- **`405` on GET** — expected; POST one JSON-RPC message.
- **Debugging a tool** — `enable_pipeline_debug` on the **tool** rule, then
  `list_pipeline_logs`/`get_pipeline_log`; the sibling runs as its own pipeline execution.

## Related skills

- **pipelines** — the handlers a tool rule chains; expression syntax
- **proxy-rules** — rule sets, ordering, attaching a set to an alias
- **rules-as-code** — the git layout, `bffless rules push/test`, CI sync
- **authorization** — global vs project roles; API keys are never admin
- **pipeline-to-skill** — turn a rule set's API into an agent skill (the admin-MCP side)
