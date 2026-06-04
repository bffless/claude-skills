---
name: authentication
description: Cross-domain authentication using the admin login relay pattern, built-in /_bffless/auth endpoints, and cookie-based sessions
---

# Authentication

BFFless authenticates users on a single admin host (`admin.<primary-domain>`) and reuses that session across every site it serves. For the common case — a primary domain plus its subdomains — the SuperTokens session cookie (`sAccessToken`) is shared on `.<primary-domain>` and works directly. For the edge case of additional cross-origin custom domains, BFFless relays the session into a per-domain `bffless_access` JWT via a one-time bounce through the admin. Both cases expose the same content-side surface at `/_bffless/auth/*` — built into BFFless nginx, no proxy rules required.

## How Authentication Works

### The Common Case: Primary Domain (+ Subdomains)

A self-hosted BFFless install has **one primary domain** (e.g., `foo.com` — the root you pick during setup). The admin lives at `admin.foo.com` and content can be served from `foo.com` itself or any subdomain (`bar.foo.com`, `app.foo.com`, etc.). Almost every install runs entirely in this mode.

All of these share `.foo.com` as a parent, so the SuperTokens session cookie (`sAccessToken`) reaches every one of them automatically. There is no `bffless_access` cookie in this mode — the session is always validated against `sAccessToken`. (BFFless multi-tenant hosting at `*.workspace.bffless.app` is mechanically the same setup, with `workspace.bffless.app` playing the role of the primary domain.)

When a user hits a private deployment on the primary domain and isn't authenticated:

1. Backend redirects to `https://admin.foo.com/login?redirect=<original-path>&tryRefresh=true`
2. The login page tries a session refresh first (the `tryRefresh` param), in case the cookie is just expired
3. If refresh fails, the user logs in normally
4. After login, the user is redirected back to the original path
5. The `sAccessToken` cookie now travels with every request to either the content or admin host

### Edge Case: Additional Cross-Origin Custom Domains (`customDomainRelay`)

A single BFFless install can also serve content from **additional registered domains** that aren't under the primary — e.g., attaching `bat.com` to a `foo.com` install. This is uncommon for OSS users; reach for it only when you genuinely need to serve content from a separately-owned root domain.

Because `bat.com` and `foo.com` are different origins, the SuperTokens cookie can't reach `bat.com`. SuperTokens itself has no multi-domain cookie support, so BFFless mints its own short-lived JWT and sets it as `bffless_access` on `bat.com` via a one-time relay through `admin.foo.com`. The `bffless_access` cookie **only exists on these cross-origin custom domains** — it is never set on the primary or any of its subdomains.

The relay flow:

1. User visits a private page on `bat.com/portal/`
2. Frontend detects no auth (via `/_bffless/auth/session`)
3. Frontend redirects to the admin login with relay params:
   ```
   https://admin.foo.com/login?customDomainRelay=true&targetDomain=bat.com&redirect=%2Fportal%2F
   ```
4. User logs in on the admin domain (or is already logged in via SuperTokens session)
5. After login, the frontend calls `POST /api/auth/domain-token` with:
   ```json
   { "targetDomain": "bat.com", "redirectPath": "/portal/" }
   ```
6. Backend validates that `targetDomain` is a registered domain for this workspace, then mints a short-lived JWT (the "domain token")
7. Backend returns a `redirectUrl` pointing to the callback on the content domain: `https://bat.com/_bffless/auth/callback?token=...&redirect=/portal/`
8. The callback endpoint validates the token, sets `bffless_access` and `bffless_refresh` HttpOnly cookies, and redirects to the original path

### Default: Use `/_bffless/auth/*`

The `/_bffless/auth/*` endpoints are **built into BFFless nginx** and handled by a dedicated controller — they work on every domain without any configuration. They are separate from the SuperTokens `/api/auth/*` endpoints (which only exist on the admin host).

Reach for `/_bffless/auth/*` first. The two situations where it isn't enough are:

- You need to **clear the SuperTokens session** (true logout) on the primary domain or one of its subdomains.
- You need an endpoint not in the built-in surface (OAuth start/callback, `session/refresh`, etc.).

For those, set up the [reverse-proxy rule](#advanced-reverse-proxy-to-supertokens-endpoints) and call the proxied `/auth/*` path. The proxy only works on the primary domain (the SuperTokens cookie has to reach the request). On an additional cross-origin custom domain like `bat.com`, stick with `/_bffless/auth/*`.

## Auth Endpoints (Built-in)

All auth endpoints are available at `/_bffless/auth/*` on any domain served by BFFless — no proxy rules needed.

| Endpoint                    | Method | Purpose                                                  |
| --------------------------- | ------ | -------------------------------------------------------- |
| `/_bffless/auth/session`    | GET    | Check current session — see response shape below         |
| `/_bffless/auth/refresh`    | POST   | Refresh an expired access token using the refresh cookie |
| `/_bffless/auth/callback`   | GET    | Exchange a domain relay token for auth cookies           |
| `/_bffless/auth/logout`     | POST   | Clear `bffless_access` / `bffless_refresh` cookies (does NOT clear SuperTokens session — see [Advanced](#advanced-reverse-proxy-to-supertokens-endpoints)) |
| `/_bffless/auth/signin`     | POST   | In-page email+password sign-in (mints `bffless_access`)  |
| `/_bffless/auth/signup`     | POST   | In-page email+password sign-up                           |
| `/_bffless/auth/forgot-password` | POST | Trigger password-reset email                          |
| `/_bffless/auth/reset-password`  | POST | Complete password reset with token                    |
| `/_bffless/auth/verify-email`    | POST | Verify email with token                               |
| `/_bffless/auth/send-verification-email` | POST | Resend the verification email             |
| `/_bffless/auth/login-methods`   | GET  | Enabled auth providers / signup gates                 |
| `/_bffless/auth/check-email`     | POST | Test if an email exists in the workspace              |

### Session Endpoint Response Shape

`GET /_bffless/auth/session` has **three** possible outcomes — make sure your client distinguishes all three:

| Outcome | Status | Body | Meaning |
| ------- | ------ | ---- | ------- |
| Logged in | `200` | `{ "authenticated": true, "user": { id, email, role } }` | Use the user object |
| **Guest** | **`200`** | **`{ "authenticated": false, "user": null }`** | **Not logged in — do NOT trust `res.ok` alone** |
| Expired | `401` | `"try refresh token"` | Call `/_bffless/auth/refresh`, then retry session |

**Common bug**: writing `if (res.ok) return res.json()` and treating guests as authenticated. The body's `authenticated` field is the source of truth, not the HTTP status.

### Session Check Priority

The `/_bffless/auth/session` endpoint checks auth in this order:

1. **`bffless_access` cookie** — domain-relay JWT issued by the callback flow (cross-origin custom domains)
2. **`sAccessToken` cookie** — SuperTokens session (fallback for shared-parent topologies: primary domain & enterprise workspace subdomains)

If the access token is expired, it returns `401` with `"try refresh token"` to signal the client should call `/_bffless/auth/refresh`.

## Advanced: Reverse-Proxy to SuperTokens Endpoints

The built-in `/_bffless/auth/*` controller covers the read path (`session`) and the in-page sign-in / sign-up / password-reset flows, but it is **not a complete proxy to the underlying SuperTokens routes**. Two things specifically are missing:

1. **It can't clear the SuperTokens session.** `/_bffless/auth/logout` only deletes the `bffless_access` / `bffless_refresh` cookies that were minted by the domain-relay callback. The real `sAccessToken` cookie lives on the parent admin domain (`.bffless.app`, `.yourdomain.com`) and is managed by SuperTokens' `signOut()`. There's no built-in endpoint on the content domain that can revoke it.
2. **It exposes a curated subset of endpoints.** OAuth callbacks, provider lists (`/api/auth/oauth/*`), `session/refresh` (the SuperTokens-format refresh, distinct from `/_bffless/auth/refresh`), and a few other internal routes are only available under the admin's `/api/auth/*` namespace.

When you need any of these from a content domain, set up a **reverse proxy rule** from a prefix path on the content domain to the admin backend's `/api/auth` namespace. This is the same pattern the admin UI itself uses.

### The Proxy Rule (canonical example)

For a workspace whose admin lives at `admin.<workspace>`, add an **External Proxy** rule to the content alias:

| Field                       | Value                                          |
| --------------------------- | ---------------------------------------------- |
| Path Pattern                | `/auth/*` (or `/api/auth/*` — choose one)      |
| Method                      | Any                                            |
| Rule Type                   | External Proxy                                 |
| Target URL                  | `http://localhost:3000/api/auth` (same-instance backend) **or** `https://admin.<workspace>/api/auth` (cross-instance) |
| Strip matched path prefix   | ON                                             |
| Preserve original Host      | OFF                                            |
| Forward cookies to target   | **ON** (required — the session cookie has to travel with the request) |

`localhost:3000` is the internal CE backend on the same node; BFFless allows HTTP targets only for `*.svc` / `localhost`. For cross-instance setups, point at the admin's HTTPS URL.

With the rule above:

```
GET  j5s.dev/auth/session    →  http://localhost:3000/api/auth/session
POST j5s.dev/auth/signout    →  http://localhost:3000/api/auth/signout
GET  j5s.dev/auth/oauth/...  →  http://localhost:3000/api/auth/oauth/...
```

### Response Shape Differs From `_bffless/auth/session`

The proxied SuperTokens session endpoint returns a **richer object** than the BFFless one — both `emailVerified` and the session handle, with `user: null` instead of `authenticated: false` for guests:

```json
// GET /auth/session  (proxied to /api/auth/session)
{
  "session": { "userId": "...", "handle": "..." },
  "user":    { "id": "...", "email": "...", "role": "admin" },
  "emailVerified": true,
  "emailVerificationRequired": false
}
```

Compare with the BFFless built-in (covered above):

```json
// GET /_bffless/auth/session
{ "authenticated": true, "user": { "id": "...", "email": "...", "role": "admin" } }
```

Pick one shape per client and stick with it; mixing causes the same "treated guest as authed" bug described earlier. If you need `emailVerified` on the content domain, use the proxied endpoint.

### When to Use Each

| Need                                                 | Use                                                        |
| ---------------------------------------------------- | ---------------------------------------------------------- |
| Cheap session check, no SuperTokens dep              | `/_bffless/auth/session`                                   |
| In-page sign-in / sign-up / forgot-password dialog   | `/_bffless/auth/signin` etc. (works on true custom domains too) |
| Custom-domain relay callback                         | `/_bffless/auth/callback` (built-in, can't be proxied)     |
| **Clearing the SuperTokens session** (real logout)   | Proxied `/auth/signout` **or** bounce through `admin.<workspace>/logout` (see [Logout](#logout)) |
| OAuth / SSO flows started from the content domain    | Proxied `/auth/oauth/*`                                    |
| `emailVerified`, session handle, pending invitations | Proxied `/auth/session`                                    |

### Caveat: This Only Works When the Cookie Reaches the Proxy

The reverse-proxy approach depends on the browser sending the SuperTokens session cookie to the content domain so BFFless can forward it. That works on the **primary domain and its subdomains** (`foo.com`, `bar.foo.com`, …) because `sAccessToken` is on `.foo.com`. It also works on `*.workspace.bffless.app` since that is mechanically the same setup.

It does **not** work on additional cross-origin custom domains (a `bat.com` attached to a `foo.com` install): the `sAccessToken` cookie never reaches `bat.com`, so the proxy has nothing to forward. Use `_bffless/auth/*` + the admin-bounce logout on those.

## Frontend Integration

### Checking Session (with automatic token refresh)

Use a shared promise pattern to avoid duplicate session checks across components:

```typescript
type Session =
  | { authenticated: true; user: { id: string; email?: string; role?: string } }
  | { authenticated: false };

async function checkSession(): Promise<Session> {
  // Reuse shared session promise so multiple components don't duplicate requests
  if (!(window as any).__bfflessSession) {
    (window as any).__bfflessSession = (async (): Promise<Session> => {
      const get = () => fetch('/_bffless/auth/session', { credentials: 'include' });

      let res = await get();
      if (res.status === 401) {
        // Token expired — try refreshing, then retry the session check
        const refreshRes = await fetch('/_bffless/auth/refresh', {
          method: 'POST',
          credentials: 'include',
        });
        if (refreshRes.ok) res = await get();
      }

      if (!res.ok) return { authenticated: false };

      // IMPORTANT: a 200 can still be a guest — the body decides.
      const body = await res.json();
      if (body?.authenticated === false || body?.user == null) {
        return { authenticated: false };
      }
      return { authenticated: true, user: body.user ?? body };
    })().catch(() => ({ authenticated: false }) as Session);
  }

  return (window as any).__bfflessSession;
}
```

The flow is: session check → if 401, refresh and retry → inspect `body.authenticated`. Do NOT treat any 200 as authenticated; the guest response is also a 200 (see the response shape table above).

### Redirecting to Login

When an unauthenticated user needs to sign in, send them to the admin login on the **primary domain** (`admin.foo.com`) and tell it where to return them. **How you encode "where to return" depends on the topology — and the two cases are not interchangeable.**

> **Do not reach for `customDomainRelay` by default.** It is only for additional cross-origin custom domains (see the second helper below). Adding it for the primary domain or one of its subdomains is wrong: the relay is unnecessary there (the `sAccessToken` cookie already reaches the host), it sets a redundant `bffless_access` cookie, and the new-user **sign-up** bounce drops the relay params — so signups get stranded on `admin.foo.com`. Use the no-relay helper for anything under your primary domain.

#### Common case: primary domain + subdomains (no relay)

`sAccessToken` is shared on `.foo.com`, so you just need the admin to bounce the browser back to the content host. Pass an **absolute** return URL in `redirect` — a relative `redirect=/` is treated as relative to the admin and leaves the user stranded on `admin.foo.com`. The admin validates the URL is within the same base domain before honoring it, then does a full-page navigation back so the shared cookie authenticates on arrival.

```typescript
function getLoginUrl(adminLoginUrl: string, redirectPath = '/'): string {
  // Absolute URL back to THIS host. `origin` includes scheme + host + port,
  // so it survives local dev. Without it, `redirect=/` lands the user on
  // admin.foo.com instead of coming back here.
  const returnTo = window.location.origin + redirectPath;
  const params = new URLSearchParams({ redirect: returnTo });
  return `${adminLoginUrl}?${params.toString()}`;
}

// Example: from app.foo.com, bounce through the admin and come back to /portal/
const session = await checkSession();
if (!session.authenticated) {
  window.location.href = getLoginUrl('https://admin.foo.com/login', '/portal/');
  // → https://admin.foo.com/login?redirect=https%3A%2F%2Fapp.foo.com%2Fportal%2F
}
```

#### Edge case: additional cross-origin custom domain (relay)

Only when content lives on a **separately-owned root** (`bat.com` attached to a `foo.com` install) does the cookie fail to reach it, so you fall back to the relay. Note the `redirect` here is a **path**, not an absolute URL — the callback lives on `targetDomain`, and the admin builds `bat.com/_bffless/auth/callback?...&redirect=<path>` for you.

```typescript
function getCustomDomainLoginUrl(adminLoginUrl: string, redirectPath = '/'): string {
  // Use `host`, NOT `hostname` — host includes the port (e.g. `localhost:5173`).
  // Using `hostname` strips the port, and the backend builds a callback URL
  // like `https://localhost/_bffless/auth/callback?...` (no port, wrong scheme)
  // which is unreachable in local dev.
  const params = new URLSearchParams({
    customDomainRelay: 'true',
    targetDomain: window.location.host,
    redirect: redirectPath, // a PATH — the callback is served on targetDomain
  });
  return `${adminLoginUrl}?${params.toString()}`;
}
```

`targetDomain` must be registered in the workspace (a `domain_mappings` entry, or a subdomain of `PRIMARY_DOMAIN`) or the `domain-token` mint is rejected. After login the admin relays through `POST /api/auth/domain-token` and redirects to `bat.com/_bffless/auth/callback`, which sets `bffless_access`.

#### Picking between them

```typescript
function loginUrlFor(adminLoginUrl: string, primaryDomain: string, redirectPath = '/') {
  const host = window.location.hostname;
  const underPrimary = host === primaryDomain || host.endsWith('.' + primaryDomain);
  return underPrimary
    ? getLoginUrl(adminLoginUrl, redirectPath)            // no relay
    : getCustomDomainLoginUrl(adminLoginUrl, redirectPath); // relay
}
```

### Logout

Logout is symmetric to login: the admin host owns the SuperTokens session, so on the primary domain (and its subdomains) you have to bounce through `admin.foo.com/logout` to actually revoke. Calling `/_bffless/auth/logout` on its own is **not enough** — it only clears `bffless_access`, which isn't even set on the primary domain. See the dedicated troubleshooting entry below.

> **Alternative for the common case:** if you've configured the [reverse-proxy rule](#advanced-reverse-proxy-to-supertokens-endpoints) (e.g., `/auth/*` → admin `/api/auth`), you can `POST /auth/signout` directly from the content domain instead of bouncing through the admin page. The proxy forwards the SuperTokens session cookie and SuperTokens clears it on `.foo.com`. The admin-bounce below is the universal fallback (and the only option on additional cross-origin custom domains, where the proxy can't reach the cookie).

```typescript
async function logout(adminLogoutUrl: string) {
  // 1. Clear the bffless_access / bffless_refresh cookies that live on this
  //    domain. No-op on the primary domain (those cookies are never set
  //    there), but required for additional cross-origin custom domains.
  try {
    await fetch('/_bffless/auth/logout', {
      method: 'POST',
      credentials: 'include',
    });
  } catch {
    // ignore — the admin bounce below is the source of truth
  }

  // 2. Bounce through the admin logout page so SuperTokens revokes the
  //    session and clears `sAccessToken` on the parent domain. The admin
  //    page validates `redirect` (same base-domain only) and sends the
  //    user back.
  const redirect = window.location.origin + window.location.pathname;
  window.location.href = `${adminLogoutUrl}?redirect=${encodeURIComponent(redirect)}`;
}

// Example:
// logout('https://admin.foo.com/logout');
```

Mirrors the login flow — same admin URL pattern, just `/logout` instead of `/login`. If you derive both URLs from a single env var, do it explicitly rather than munging the login URL with regex, so the intent is obvious to the next reader.

### Updating UI Based on Auth State (Header example)

```typescript
// Check auth state and update Login/Portal links
window.__bfflessSession = window.__bfflessSession || checkBfflessSession().catch(() => null);

window.__bfflessSession.then((data) => {
  if (data?.authenticated) {
    // User is logged in — update nav links
    document.querySelectorAll('[data-auth-link]').forEach((el) => {
      el.textContent = 'Portal';
    });
  }
});
```

## Local Development

There is no auth backend running on `localhost`, so `/_bffless/auth/*` 404s out of the box. There are two patterns for working around this:

### 1. Proxy `/_bffless` to a deployed workspace (real auth)

Best when you want to exercise the real cookie/relay flow. Configure your dev server to proxy `/_bffless` (and usually `/api`) to a real BFFless deployment:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': { target: 'https://foo.com', changeOrigin: true },
      '/_bffless': { target: 'https://foo.com', changeOrigin: true },
    },
  },
});
```

Caveats:
- Login redirects you to the admin domain. The `targetDomain` you send must be a registered domain in that workspace — `localhost:5173` will get rejected with "Domain not registered" unless an admin adds it (or you point at a workspace that does). Most teams add their dev host to a sandbox workspace's domain mappings for this purpose.
- The session endpoint will return the **guest** shape (`200 { authenticated: false, user: null }`) until you complete the login + callback round trip, which is why the body-inspection pattern above is required.

### 2. Mock `/_bffless/auth/*` with MSW (no backend)

Best when you want to iterate on auth-gated UI without leaving localhost. [MSW](https://mswjs.io) intercepts at the service-worker layer so the production `fetch` calls stay untouched:

```ts
// src/mocks/handlers.ts
import { http, HttpResponse, passthrough } from 'msw';

const STORAGE_KEY = 'bffless:mockAuth';

function readMock() {
  try { return JSON.parse(localStorage.getItem(STORAGE_KEY) ?? '{}'); }
  catch { return {}; }
}

export const handlers = [
  http.get('/_bffless/auth/session', () => {
    const m = readMock();
    if (!m.enabled) return passthrough();
    if (!m.authenticated) {
      return HttpResponse.json({ authenticated: false, user: null });
    }
    return HttpResponse.json({ authenticated: true, user: m.user });
  }),
  http.post('/_bffless/auth/refresh', () => {
    const m = readMock();
    if (!m.enabled) return passthrough();
    return new HttpResponse(null, { status: m.authenticated ? 200 : 401 });
  }),
  http.post('/_bffless/auth/logout', () => {
    const m = readMock();
    if (!m.enabled) return passthrough();
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ ...m, authenticated: false }));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

Then boot the worker only in dev, before render:

```ts
// src/main.tsx
async function enableMocks() {
  if (!import.meta.env.DEV) return;
  const { setupWorker } = await import('msw/browser');
  const { handlers } = await import('./mocks/handlers');
  await setupWorker(...handlers).start({ onUnhandledRequest: 'bypass' });
}
enableMocks().then(() => { /* createRoot(...).render(...) */ });
```

Add a small dev-only panel (rendered when `import.meta.env.DEV`) that writes `{ enabled, authenticated, user }` to localStorage and dispatches a `CustomEvent` so your session hook can refetch. With this setup you can toggle authed/guest and swap user attributes live without restarting the dev server.

Caveats:
- Returning `passthrough()` when `enabled === false` lets you fall back to a proxied real backend (pattern #1) on demand.
- MSW requires `public/mockServiceWorker.js`; generate it once with `npx msw init public/ --save`.

## Auth Flow Diagram

```
Additional Custom Domain Flow (edge case):
┌──────────────────┐     JS redirect        ┌──────────────────────────┐
│  bat.com         │ ──────────────────→    │  admin.foo.com/login     │
│  (private page)  │  customDomainRelay=    │  ?customDomainRelay=true │
│                  │  true&targetDomain=    │  &targetDomain=bat.com   │
└──────────────────┘  bat.com               └────────────┬─────────────┘
        ▲                                                │
        │                                     User logs in (SuperTokens)
        │                                                │
        │                                                ▼
        │                                   POST /api/auth/domain-token
        │                                   → returns { token, redirectUrl }
        │                                                │
        │              302 redirect                      │
        │  ←─────────────────────────────────────────────┘
        │  to: bat.com/_bffless/auth/callback?token=...
        │
        ▼
┌──────────────────┐
│  /_bffless/auth  │  Validates token, sets bffless_access
│  /callback       │  + bffless_refresh cookies
│  (built-in)      │  → 302 redirect to /portal/
└──────────────────┘
```

## Troubleshooting

**User gets stuck in a redirect loop on an additional custom domain (e.g., `bat.com`)?**

- **Most common cause:** Calling `/api/auth/session` instead of `/_bffless/auth/session`. The relay flow sets `bffless_access`, which only `/_bffless/auth/*` recognizes — and `/api/auth/*` doesn't even exist on `bat.com` without a reverse-proxy rule (which wouldn't help anyway, since the `sAccessToken` cookie can't reach `bat.com`).
- Verify the custom domain is registered in `domain_mappings` with `isActive = true`
- Ensure cookies are being set (requires HTTPS for `Secure` flag)

**"Domain not registered" error on domain-token?**

- The `targetDomain` must match a `domain_mappings` entry or be a subdomain of `PRIMARY_DOMAIN`
- Check for www vs non-www mismatch

**After login the user lands on `admin.foo.com` instead of the content site?**

- Two causes, both in how the login link was built (see [Redirecting to Login](#redirecting-to-login)):
  1. `redirect` was a relative path (`redirect=/`). The admin honors it relative to itself. Pass an **absolute** URL (`https://app.foo.com/...`) for the no-relay common case.
  2. `customDomainRelay=true` was added for a subdomain of the primary domain. The relay is for cross-origin custom domains only; on a primary subdomain, drop it and use the absolute-URL `redirect` instead.
- Same root cause strands **new sign-ups**: the relay params are not carried through the sign-up bounce, so a user who clicks "Sign up" from a relay login URL finishes on the admin. The no-relay helper avoids this entirely.

**Session check returns 401 but user just logged in?**

- On the primary domain or one of its subdomains: verify `COOKIE_DOMAIN` is set to `.foo.com` so `sAccessToken` is shared across them
- On an additional cross-origin custom domain (`bat.com` attached to a `foo.com` install): verify the `/_bffless/auth/callback` was reached and `bffless_access` was set
- Check that the `bffless_access` or `sAccessToken` cookie is present in the request

**Logout returns 200 "Logged out successfully" but the next session check still returns `authenticated: true`?**

This is the most-reported logout footgun on the primary domain. `/_bffless/auth/logout` only clears `bffless_access` / `bffless_refresh`, but on the primary domain those cookies never existed — the session is in `sAccessToken` on `.foo.com`, which `/_bffless/auth/logout` cannot touch.

Fix: either configure the [reverse-proxy rule](#advanced-reverse-proxy-to-supertokens-endpoints) and `POST /auth/signout` directly, or navigate to `admin.foo.com/logout?redirect=<current-page>`. The admin page calls SuperTokens `signOut()`, which revokes the session and clears the shared cookie, then redirects back. See the [Logout](#logout) section for the full pattern.

On a cross-origin custom domain (`bat.com`) `/_bffless/auth/logout` actually does clear the relevant cookies (`bffless_access` / `bffless_refresh`) — no admin bounce required there.
