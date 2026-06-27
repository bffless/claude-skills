---
name: proxy-rules
description: Forward requests to backend APIs without CORS
---

# Proxy Rules

**Docs**: https://docs.bffless.app/features/proxy-rules/

Proxy rules forward requests from your static site to backend APIs, eliminating CORS issues and hiding API endpoints from clients.

## Configuration Levels

**Project defaults**: Apply to all aliases unless overridden

**Alias overrides**: Specific aliases can have their own proxy rule sets assigned

## Rule Sets

Proxy rules are organized into **rule sets** — named, reusable groups of rules.

An alias can have **multiple rule sets** attached. When multiple rule sets are assigned, their rules are merged in priority order (first set wins if two sets define the same path+method).

This allows logical grouping, e.g.:
- `api-proxy` — routes to your backend API
- `pipelines` — pipeline-based handlers for chat, forms, etc.
- `auth-proxy` — cookie-to-bearer token transformation

### Assigning Rule Sets to Aliases

Use `update_alias` with `proxyRuleSetIds` (array) to attach one or more rule sets:

```
update_alias(
  repository: "owner/repo",
  alias: "production",
  proxyRuleSetIds: ["rule-set-id-1", "rule-set-id-2"]
)
```

The legacy `proxyRuleSetId` (singular) still works for backwards compatibility but only supports one rule set.

**Important**: After creating proxy rules, you must assign the rule set(s) to an alias for rules to take effect. Rules won't be active until linked to an alias.

## Rule Structure

Each rule specifies:
- **Path pattern**: Which requests to intercept
- **Target**: Backend URL to forward to
- **Strip prefix**: Whether to remove the matched prefix

## Pattern Types

**Prefix match** (most common):
```
/api/* → https://api.example.com
```
Request to `/api/users` forwards to `https://api.example.com/users`

**Exact match**:
```
/health → https://backend.example.com/status
```
Only exact path matches, no wildcards

**Suffix match**:
```
*.json → https://data.example.com
```
Matches any path ending in `.json`

## Strip Prefix Behavior

With strip prefix ON (default):
```
/api/* → https://backend.com
/api/users → https://backend.com/users
```

With strip prefix OFF:
```
/api/* → https://backend.com
/api/users → https://backend.com/api/users
```

## Common Patterns

**API proxy**: `/api/*` → your backend server

**Third-party API**: `/stripe/*` → `https://api.stripe.com` (hides API from client)

**Microservices**: Different prefixes route to different services

## Authoring Handler Code (function_handler)

Proxy rule sets can include pipeline rules that carry a `function_handler` (custom JavaScript for transformation/logic). When you create or update such a rule through the MCP, the `code` string inside `function_handler.config` is stored **verbatim** and displayed verbatim in the admin UI — the UI does not reformat it.

**Always emit multi-line, indented source — never a minified one-liner.**

- Use real newlines (`\n` in the JSON payload) between statements.
- Indent with 2 spaces, one statement per line — don't chain statements with semicolons onto one line.
- Applies to any non-trivial `function_handler` body. A true one-liner like `function handler() { return {}; }` is fine; everything else should be expanded.

Bad (do not submit code like this):

```json
{ "code": "function handler({ request }) { var h = request.headers || {}; var ip = (h['x-forwarded-for']||'').split(',')[0].trim() || 'unknown'; return { ip: ip }; }" }
```

Good:

```json
{
  "code": "function handler({ request }) {\n  var headers = (request && request.headers) || {};\n  var xff = headers['x-forwarded-for'] || '';\n\n  var firstIp = '';\n  if (typeof xff === 'string' && xff.length > 0) {\n    var parts = xff.split(',');\n    firstIp = parts[0] ? parts[0].trim() : '';\n  }\n\n  return {\n    ip: firstIp || 'unknown',\n  };\n}\n"
}
```

The user opens these rules in the admin UI to review and edit them later — a wall-of-text `code` field forces them to manually reformat before they can read it. See the **pipelines** skill ("Authoring Handler Code via MCP") for the full handler authoring guidance and sandbox constraints.

## Security Notes

- All proxy targets must use HTTPS
- BFFless validates targets to prevent SSRF attacks
- Headers like `Host` are rewritten to match target
- Client IP forwarded via `X-Forwarded-For`

## Troubleshooting

**Requests not proxying?**
- Check path pattern matches request URL exactly
- Verify rule set is assigned to the alias via `update_alias`
- Confirm target URL is HTTPS and reachable

**Getting CORS errors still?**
- Ensure the proxy rule set is assigned to the alias being accessed
- Check browser DevTools for actual request URL (may not be hitting proxy)
