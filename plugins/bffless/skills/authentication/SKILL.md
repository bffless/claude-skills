---
name: authentication
description: Cross-domain authentication using the admin login relay pattern, built-in /_bffless/auth endpoints, and cookie-based sessions
---

# Authentication

BFFless uses a cross-domain authentication relay pattern. Users authenticate at the workspace's admin domain (`admin.<workspace>`) and are relayed back to the content domain with auth cookies. Auth endpoints on the content domain are accessed via the **built-in `/_bffless/auth/*` endpoints** — no proxy rules required.

## How Authentication Works

### Workspace Subdomains

For workspace subdomains (e.g., `myalias.sandbox.workspace.bffless.app`), SuperTokens session cookies (`sAccessToken`) work directly because they share the parent domain.

When a user visits a private deployment and isn't authenticated:

1. Backend redirects to `https://admin.<workspace>/login?redirect=<original-path>&tryRefresh=true`
2. The login page attempts a session refresh first (the `tryRefresh` param)
3. If refresh fails, the user logs in normally
4. After login, the user is redirected back to the original path
5. The `sAccessToken` cookie is valid across all subdomains of the workspace

### Custom Domains (customDomainRelay)

For custom domains (e.g., `www.bffless.com`), SuperTokens cookies don't work because they're on a completely different domain. BFFless uses a **domain relay** flow:

1. User visits a private page on `www.bffless.com/portal/`
2. Frontend detects the user is not authenticated (via `/_bffless/auth/session`)
3. Frontend redirects to the admin login with relay params:
   ```
   https://admin.console.bffless.app/login?customDomainRelay=true&targetDomain=www.bffless.com&redirect=%2Fportal%2F
   ```
4. User logs in on the admin domain (or is already logged in via SuperTokens session)
5. After login, the frontend calls `POST /api/auth/domain-token` with:
   ```json
   { "targetDomain": "www.bffless.com", "redirectPath": "/portal/" }
   ```
6. Backend validates that `targetDomain` is a registered domain for this workspace, then creates a short-lived JWT (the "domain token")
7. Backend returns a `redirectUrl` pointing to the callback on the custom domain: `https://www.bffless.com/_bffless/auth/callback?token=...&redirect=/portal/`
8. The callback endpoint validates the token, sets `bffless_access` and `bffless_refresh` HttpOnly cookies, and redirects to the original path

### Important: Use `/_bffless/auth/*`, NOT `/api/auth/*`

The `/_bffless/auth/*` endpoints are **built into BFFless nginx** and handled by a dedicated controller. They are separate from the SuperTokens `/api/auth/*` endpoints. Do NOT use `/api/auth/*` on custom domains — those are SuperTokens endpoints that use different cookies (`sAccessToken`) which are not set by the domain relay flow.

The domain relay callback sets `bffless_access` and `bffless_refresh` cookies, which are only recognized by the `/_bffless/auth/*` endpoints. Using `/api/auth/session` instead of `/_bffless/auth/session` will cause a redirect loop because the SuperTokens session check won't find the `bffless_access` cookie.

## Auth Endpoints (Built-in)

All auth endpoints are available at `/_bffless/auth/*` on any domain served by BFFless — no proxy rules needed.

| Endpoint                    | Method | Purpose                                                  |
| --------------------------- | ------ | -------------------------------------------------------- |
| `/_bffless/auth/session`    | GET    | Check current session — see response shape below         |
| `/_bffless/auth/refresh`    | POST   | Refresh an expired access token using the refresh cookie |
| `/_bffless/auth/callback`   | GET    | Exchange a domain relay token for auth cookies           |
| `/_bffless/auth/logout`     | POST   | Clear auth cookies                                       |

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

1. **`bffless_access` cookie** — custom domain JWT issued by the callback flow
2. **`sAccessToken` cookie** — SuperTokens session (fallback for workspace subdomains)

If the access token is expired, it returns `401` with `"try refresh token"` to signal the client should call `/_bffless/auth/refresh`.

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

When unauthenticated, redirect the user to the admin login with relay params. Use the **promoted admin domain** (e.g., `admin.console.bffless.app`), not the full workspace subdomain:

```typescript
function getLoginUrl(adminLoginUrl: string, redirectPath: string): string {
  // Use `host`, NOT `hostname` — host includes the port (e.g. `localhost:5173`).
  // Using `hostname` strips the port, and the backend builds a callback URL
  // like `https://localhost/_bffless/auth/callback?...` (no port, wrong scheme)
  // which is unreachable in local dev.
  const targetDomain = window.location.host;
  const params = new URLSearchParams({
    customDomainRelay: 'true',
    targetDomain,
    redirect: redirectPath,
  });
  return `${adminLoginUrl}?${params.toString()}`;
}

// Example: redirect to admin login, then relay back to /portal/
const session = await checkSession();
if (!session) {
  window.location.href = getLoginUrl(
    'https://admin.console.bffless.app/login',
    '/portal/',
  );
}
```

### Logout

Logout is symmetric to login: the admin domain owns the SuperTokens session, so you have to bounce through it to actually revoke. Calling `/_bffless/auth/logout` on its own is **not enough** on workspace subdomains — see the dedicated troubleshooting entry below.

```typescript
async function logout(adminLogoutUrl: string) {
  // 1. Clear the bffless_access / bffless_refresh cookies that live on this
  //    domain. No-op on workspace subdomains (those cookies are never set
  //    there), but required for custom domains.
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
// logout('https://admin.console.bffless.app/logout');
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
      '/api': { target: 'https://yourworkspace.bffless.app', changeOrigin: true },
      '/_bffless': { target: 'https://yourworkspace.bffless.app', changeOrigin: true },
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
Custom Domain Flow:
┌──────────────────┐     JS redirect        ┌──────────────────────────┐
│  www.bffless.com │ ──────────────────→    │  admin.<workspace>/login │
│  (private page)  │  customDomainRelay=    │  ?customDomainRelay=true │
│                  │  true&targetDomain=    │  &targetDomain=www...    │
└──────────────────┘  www.bffless.com       └────────────┬─────────────┘
        ▲                                                │
        │                                     User logs in (SuperTokens)
        │                                                │
        │                                                ▼
        │                                   POST /api/auth/domain-token
        │                                   → returns { token, redirectUrl }
        │                                                │
        │              302 redirect                      │
        │  ←─────────────────────────────────────────────┘
        │  to: www.bffless.com/_bffless/auth/callback?token=...
        │
        ▼
┌──────────────────┐
│  /_bffless/auth  │  Validates token, sets bffless_access
│  /callback       │  + bffless_refresh cookies
│  (built-in)      │  → 302 redirect to /portal/
└──────────────────┘
```

## Troubleshooting

**User gets stuck in a redirect loop?**

- **Most common cause:** Using `/api/auth/session` instead of `/_bffless/auth/session`. The domain relay callback sets `bffless_access` cookies which are only recognized by `/_bffless/auth/*` endpoints. The `/api/auth/*` endpoints check SuperTokens cookies (`sAccessToken`) which are NOT set by the domain relay flow.
- Verify the custom domain is registered in `domain_mappings` with `isActive = true`
- Ensure cookies are being set (requires HTTPS for `Secure` flag)

**"Domain not registered" error on domain-token?**

- The `targetDomain` must match a `domain_mappings` entry or be a subdomain of `PRIMARY_DOMAIN`
- Check for www vs non-www mismatch

**Session check returns 401 but user just logged in?**

- On custom domains: verify the `/_bffless/auth/callback` was reached and cookies were set
- On workspace subdomains: verify `COOKIE_DOMAIN` is configured for cross-subdomain cookie sharing
- Check that the `bffless_access` or `sAccessToken` cookie is present in the request

**Admin login URL — use promoted domain, not workspace subdomain:**

- If the workspace has a promoted domain (e.g., `console.bffless.app`), use `admin.console.bffless.app`, NOT `admin.console.workspace.bffless.app`
- The workspace subdomain format still works but the promoted domain is cleaner

**Logout returns 200 "Logged out successfully" but the next session check still returns `authenticated: true`?**

This is the most-reported logout footgun. It means you called `/_bffless/auth/logout` alone:

- `/_bffless/auth/logout` only clears the **custom-domain JWT cookies** (`bffless_access`, `bffless_refresh`).
- The session endpoint falls back to the **SuperTokens session** (`sAccessToken`) when no `bffless_access` cookie is present. That cookie lives on the parent domain (`.bffless.app` / `.yourdomain.com`) and was set by the admin login — it is not cleared by `/_bffless/auth/logout`.
- On workspace subdomains the `bffless_access` cookie was never set in the first place, so `/_bffless/auth/logout` is effectively a no-op and the SuperTokens fallback re-authenticates the user immediately.

Fix: after the `/_bffless/auth/logout` call, navigate to `admin.<workspace>/logout?redirect=<current-page>`. The admin page calls SuperTokens `signOut()`, which revokes the session and clears the shared cookie, then redirects back. See the [Logout](#logout) section for the full pattern.
