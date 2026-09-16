# Cross-Subdomain Session Sharing (Supabase Auth)

Makes logging into one product (or the Hub) keep the person logged into
every other product on `*.roi-engg.com`, instead of each subdomain holding
its own isolated session.

**Apply this identically to every product** (Hub, Cuplok Estimator,
BEAM//FE, Table Leg Calculator, Aluma Frame Estimator). If even one product
is left on the SDK's default storage while the others switch, that one
product won't share the session with the rest.

## 1. Cookie storage adapter

Add near your existing `SUPABASE_URL` / `SUPABASE_ANON_KEY` constants,
**before** `supabaseClient` is created:

```js
// Cookie storage scoped to .roi-engg.com so the Supabase session is
// visible across every subdomain (Hub, Cuplok, BEAM//FE, etc.) instead
// of being isolated per-origin like localStorage is. Only meaningful in
// PROD — localhost isn't a subdomain of roi-engg.com, so DEV keeps using
// the browser default (localStorage) automatically.
const COOKIE_DOMAIN = ".roi-engg.com";
const COOKIE_MAX_AGE = 60 * 60 * 24 * 7; // 7 days — match/adjust to your refresh token lifetime

function setCookie(name, value, maxAge) {
  const parts = [
    `${name}=${encodeURIComponent(value)}`,
    "path=/",
    `domain=${COOKIE_DOMAIN}`,
    `max-age=${maxAge}`,
    "samesite=Lax"
  ];
  if (location.protocol === "https:") parts.push("secure");
  document.cookie = parts.join("; ");
}
function getCookie(name) {
  const escaped = name.replace(/[.$?*|{}()[\]\\/+^]/g, "\\$&");
  const match = document.cookie.match(new RegExp("(?:^|; )" + escaped + "=([^;]*)"));
  return match ? decodeURIComponent(match[1]) : null;
}
function deleteCookie(name) {
  document.cookie = `${name}=; path=/; domain=${COOKIE_DOMAIN}; max-age=0`;
}

const crossSubdomainCookieStorage = {
  getItem: key => getCookie(key),
  setItem: (key, value) => setCookie(key, value, COOKIE_MAX_AGE),
  removeItem: key => deleteCookie(key)
};
```

## 2. Client creation

Replace your existing:

```js
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

with:

```js
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  auth: {
    storage: IS_LOCAL_DEV ? undefined : crossSubdomainCookieStorage, // undefined = SDK default (localStorage)
    persistSession: true,
    autoRefreshToken: true
  }
});
```

(Assumes each product already has an `IS_LOCAL_DEV` flag from the
localhost/127.0.0.1/`.app.github.dev` hostname check used across the
suite. If a product doesn't have one yet, add it first.)

## 3. Sign-out fix (unrelated bug, same session code path)

`auth.signOut()` defaults to `scope: 'global'`, which tries a
server-side revoke that currently 403s against the newer
`sb_publishable_...` key format — a known Supabase limitation, not a bug
in this code. Use local scope instead:

```js
await supabaseClient.auth.signOut({ scope: "local" });
```

## Rollout notes

- **DEV is unaffected on purpose** — a `.roi-engg.com`-scoped cookie is
  never sent to `localhost`, so DEV keeps using plain localStorage via
  the `IS_LOCAL_DEV` check.
- **Cookie size** — the Supabase session payload (access token + refresh
  token + user metadata) can approach the ~4KB browser cookie limit.
  After rolling this out, log into PROD for real and check the cookie
  isn't silently truncated (DevTools → Application → Cookies).
- **All-or-nothing per product** — a product still on localStorage will
  not see sessions set by a product on the cookie, and vice versa.
