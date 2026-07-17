---
name: traffic-splitting
description: Distribute traffic across aliases with weights and rules
---

# Traffic Splitting

**Docs**: https://docs.bffless.app/features/traffic-splitting/

Traffic splitting distributes requests across multiple deployment aliases. Use it for A/B testing, canary deployments, and personalized experiences.

## Configuration

### Weights

Assign percentage weights to aliases. Weights must sum to 100.

```
production: 90%
canary: 10%
```

### Sticky Sessions

When enabled, visitors consistently see the same variant. Controlled via cookie. Disable for true random distribution per request.

### Traffic Rules

Rules override weights based on conditions. Evaluated in order, first match wins.

**Rule conditions:**
- **Headers**: Match request headers (e.g., `X-Beta-User: true`)
- **Query params**: Match URL parameters (e.g., `?variant=new`)
- **Cookies**: Match cookie values
- **Share link**: Route specific share links to specific aliases

## Configuring via MCP

If the BFFless MCP server is connected, configure splits and rules directly with these tools instead of the admin UI or raw REST calls. All operate on a **domain mapping ID** (get one from `list_domains` / `get_domain`).

**Weights**
- `get_traffic_config(domainId)` — current split + sticky-session settings (`{ weights[], stickySessionsEnabled, stickySessionDuration }`); empty weights = single-alias
- `list_traffic_aliases(domainId)` — aliases available to weight (discover valid names first)
- `set_traffic_weights(domainId, weights[], stickySessionsEnabled?, stickySessionDuration?)` — set the split. `weights` is `[{ alias, weight, path? }]` and **must sum to 100** (rejected otherwise). Per-weight `path` is optional and falls back to the domain mapping's path — omit it unless a variant serves from a different subdirectory.
- `clear_traffic_weights(domainId)` — remove the split, return to the single alias

**Rules** (override weights; first match wins by priority, lower first)
- `list_traffic_rules(domainId)`
- `create_traffic_rule(domainId, alias, conditionType, conditionKey, conditionValue, priority?, label?)` — `conditionType` is `query_param` | `cookie` | `header`
- `update_traffic_rule(ruleId, ...)`
- `delete_traffic_rule(ruleId)`

Example — 50/50 production/red split with a `?v=red` force rule:
```
set_traffic_weights("<domainId>", [
  { alias: "production", weight: 50 },
  { alias: "red", weight: 50 },
])
create_traffic_rule("<domainId>", "red", "query_param", "v", "red", 10, "Force red")
```

## Use Cases

**A/B Testing**: Split traffic 50/50 between variants, measure conversion

**Canary Deployment**: Send 5% to new version, monitor for errors, gradually increase

**Beta Features**: Route users with beta header to feature branch

**Personalized Demos**: Use share links to show customized content per client

## Configuration Tips

1. Start canary deployments at 5-10%, increase gradually
2. Enable sticky sessions for consistent user experience
3. Use rules for deterministic routing (internal testing, specific clients)
4. Combine with share links for per-recipient personalization

## Troubleshooting

**Unexpected routing?**
- Rules are evaluated before weights; check rule order
- Clear cookies if sticky session is routing to old variant
- Verify alias names match exactly (case-sensitive)

**Traffic percentages seem off?**
- Small sample sizes show high variance; need 100+ requests for accuracy
- Sticky sessions can skew percentages if users have different visit frequencies
