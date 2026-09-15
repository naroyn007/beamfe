## SSO HANDSHAKE — PLATFORM HUB ↔ BEAM//FE

### Current Status

The Platform Hub is the central authentication/launcher for the engineering tools.

Platform Hub and BEAM//FE use the **same Supabase projects**:

* Development: `https://xhjqnkpsuyzodgnhhuwb.supabase.co`
* Production: `https://swhilchfrytknrbjhrlj.supabase.co`

### Platform Hub SSO Bridge

`auth-confirm.html` already implements the cross-subdomain session cookie bridge.

On successful authentication or token refresh:

```js
function writeSessionCookies(session) {
  if (!canBridge || !session) return;
  const maxAge = 60 * 60 * 24 * 30;
  document.cookie = `sb-access-token=${session.access_token}; Domain=${COOKIE_DOMAIN}; Path=/; Max-Age=${maxAge}; SameSite=Lax; Secure`;
  document.cookie = `sb-refresh-token=${session.refresh_token}; Domain=${COOKIE_DOMAIN}; Path=/; Max-Age=${maxAge}; SameSite=Lax; Secure`;
}
```

The bridge listens for:

```js
supabaseClient.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_IN' || event === 'TOKEN_REFRESHED') {
    writeSessionCookies(session);
  } else if (event === 'SIGNED_OUT') {
    clearSessionCookies();
  }
});
```

The cookie domain is `.roi-engg.com`.

### BEAM//FE SSO TODO

BEAM//FE currently has its own Supabase Auth session and direct email/password login.

It must be updated to consume the Platform Hub cookies on production:

```text
sb-access-token
sb-refresh-token
```

Then establish the Supabase session using:

```js
await sb.auth.setSession({
  access_token,
  refresh_token
});
```

After `setSession()`, the existing BEAM//FE license flow should remain unchanged:

```text
SSO session
    ↓
checkLicenseAndRoute(userEmail)
    ↓
valid Beam Analysis license?
    ├─ YES → open calculator
    └─ NO  → existing license-needed screen
```

### Important Requirements

1. **Do not remove direct email/password login.**
   It remains the fallback, especially for localhost development.

2. **Do not modify Platform Hub's existing cookie bridge unless necessary.**
   The bridge is already working conceptually.

3. **Do not expose or log access/refresh tokens.**

4. **Do not use `.roi-engg.com` cookies for localhost.**
   Localhost should continue using the normal direct-login flow.

5. Production SSO should work across:

   * Platform Hub
   * BEAM//FE
   * Aluma Frame
   * Table Leg Calculator
   * Cuplok

6. The receiving calculator must still perform its own product-specific license check. Authentication ≠ product license.

### Current BEAM//FE Auth Code

BEAM//FE production Supabase:

```js
const SUPABASE_URL =
  'https://swhilchfrytknrbjhrlj.supabase.co';
```

BEAM//FE development Supabase:

```js
const SUPABASE_URL =
  'https://xhjqnkpsuyzodgnhhuwb.supabase.co';
```

Existing license function:

```js
async function checkLicenseAndRoute(userEmail) {
  const result = await callVerifyLicense('check_license');

  if (result.valid) {
    currentLicense = {
      license_type: result.license_type,
      expires_at: result.expires_at
    };
    currentUserEmail = userEmail;
    hideLicenseGate();
    renderLicenseFooter();
    return true;
  }

  // existing license-needed handling remains unchanged
}
```

### Next Development Step

Modify only BEAM//FE `index.html`.

Add an SSO bootstrap before/around the existing:

```js
const { data } = await sb.auth.getSession();
```

The bootstrap should:

1. Detect production `roi-engg.com`.
2. Read the two SSO cookies.
3. If both exist, call `sb.auth.setSession()`.
4. Allow Supabase to refresh the session normally.
5. Run the existing `checkLicenseAndRoute()`.
6. Fall back to the existing login screen if no SSO cookies/session exist.
7. Never print token values to console.

Do **not** change the license database or license Edge Function as part of this SSO handshake unless testing proves it is required.

### Git Safety

Current BEAM//FE branch:

```text
development
```

Remote:

```text
origin/development
```

The previous uppercase/lowercase branch issue has been resolved. Do not force-push or recreate the branch.

Untracked `.vscode/` exists locally and should not be changed unless specifically needed.
