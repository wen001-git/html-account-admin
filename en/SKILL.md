---
name: html-account-admin
user-invocable: true
description: Use when a pure-frontend HTML project (single-file .html + static hosting) needs accounts / paywalls / member state and you won't spin up a backend or use JWT. The canonical pattern: store {u, sha256(salt+password)} in a repo-root accounts.json, fetch + rehash for login, sync across tabs via BroadcastChannel + localStorage. **Do NOT use** when you already have a real backend / want OIDC SSO / plan to use Auth0 Supabase Clerk Firebase / need audit logs at scale / require server-side token revocation — those go through standard identity providers. This skill ships a "zero-backend toy paywall", not a real auth system.
---

# Zero-Backend Account Admin for Static HTML Projects

> **Purpose:** Wrap the pattern behind PoemGraph's `account-admin.html` into a reusable skill, so any single-file static HTML project (PoemGraph, WhiteBoard, your next idea) can plug it in. **Audience:** (1) developers adding accounts/paywalls/member gates to single-file HTML; (2) AI coding tools implementing the integration. **How to read:** skim the TL;DR table for fit, jump to the 5 method sections, finish with the 8-step checklist.

## TL;DR

This skill answers a narrow question: **"I only have static hosting (Render / Cloudflare Pages / GitHub Pages). I want accounts / paywall / member gate on a single-file HTML, and I will NOT spin up a backend."** The core is WebCrypto + file storage — your browser does `sha256(salt + password)`, account hashes live in a repo-root `accounts.json`, the main app `fetch`es it and rehashes to log in. Cross-tab live config uses `BroadcastChannel + localStorage` as a double-trigger.

| Dimension | This skill (PoemGraph route) | Real backend auth |
|---|---|---|
| Server needed | **No** (static hosting is enough) | Required (DB + API + TLS) |
| Password storage | `sha256(salt + password)`, one-way | bcrypt/argon2, server-side |
| Session / token | `localStorage.unlocked=true` (dev tools can flip it — **intentionally weak**) | JWT/cookie + server-side revocation |
| Multi-device mgmt | None (any device that logs in is "authorized") | Per-account N devices |
| Audit / logs | None | Full server-side records |
| Best for | 10–10⁴ users: student projects, small tools, paywall PoCs | Anything that genuinely needs auth |
| Deploy | `git push` and done | Server CI/CD + secret management |

> **Honest disclosure:** this skill is a paywall, not auth. Any login state can be bypassed with browser dev tools — so don't use it to guard real secrets or real money. The value is "make it inconvenient for ordinary users", not "block attackers".

## Scope (figure out which tier you're in before using)

### Strongly recommended when

- Single-file HTML + static hosting (Render / CF Pages / Vercel / GitHub Pages / EdgeOne Pages).
- Small user base (< few hundred), paywall + content unlock, acceptable if bypass probability is low.
- You've decided against backend (or even against a single Node Function).
- The project already has an "admin" concept (typically yourself) — this skill is **not** for self-service user signup; all accounts are issued by the admin.

### Use with judgment when

- You want optional device binding / remote logout — Node Functions escape hatch (Section 5), only when you need it.
- You want "membership expires" semantics — schema has `paidUntil`, check `<= now` and you're set, ~8 lines in the main app.
- Cross-subdomain deployments (`admin.app.com` manages `app.com`) — must follow the `accountUrl=` / meta same-origin convention; otherwise CORS bites.

### Don't use / be careful when

- **Any "real auth" requirement:** banking, medical, enterprise internal — use the right-hand column of the table above.
- **Self-service signup + email verification:** this skill has zero email/OTP support.
- **GDPR / privacy-strict scenarios:** `accounts.json` is a public JSON (hashes are data too); deploying it to a CDN exposes "user exists" facts.
- **School / enterprise "account systems":** bypass is too easy; teachers can't see who logged in.
- **Anything that could land in the news as "AI / data / finance":** the paywall gives a false sense of security; don't create that illusion.

## 1. Account storage: sha256 + salt + JSON file

**State:** admin HTML computes hashes right in the browser via `crypto.subtle.digest('SHA-256', new TextEncoder().encode(salt + password))`, writes the `{u, h}` array as JSON; the admin page itself is static — **no backend connection**.

**Approach:** minimal schema. Username `u`, hash `h`, plus `version` (for schema upgrades) and one global `salt` (shared by every account). **Do not** put expiry, role, or notes in this file — store non-essential fields in the main app's IndexedDB cache. The less the accounts file changes, the better.

**Drop-in reference** (shared by admin HTML and main app):

```js
// 8 lines: hash + verify
async function hashPassword(salt, password) {
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(salt + password));
  return Array.from(new Uint8Array(buf)).map(x => x.toString(16).padStart(2, '0')).join('');
}
async function verifyPassword(salt, password, expectedHash) {
  return (await hashPassword(salt, password)) === expectedHash.toLowerCase();
}
```

**Schema (use as-is):**

```json
{
  "version": 1,
  "salt": "<?SALT?>",
  "accounts": [
    { "u": "alice", "h": "<64-char lowercase hex sha256>" },
    { "u": "bob",   "h": "<64-char lowercase hex sha256>" }
  ]
}
```

**Verification:**

- Open the browser console and run `hashPassword('<?SALT?>', 'alice')`. Compare against admin HTML's `h`; must match byte-for-byte.
- Add an account via the admin UI → download `accounts.json` → open with any JSON tool → `h` must be exactly 64 lowercase hex chars, no `0x` prefix.

## 2. HTML-side login: fetch + rehash

**State:** the main app logs in by `fetch(ACCOUNT_URL)` → get JSON → recompute hash in-browser → compare. **No token** — login state is `localStorage[<?APP_KEY?>_unlocked]` + username.

**Approach:** URL resolution follows a 3-tier priority (covers the 3 most common deployment shapes):

```js
// 11 lines: URL resolution + file:// fallback
var ACCOUNT_URL = (function(){
  try {                                  // (1) URL param wins (author explicit override)
    var u = new URL(location.href).searchParams.get('accountUrl');
    if (u) return u.trim();
  } catch(_e) {}
  var meta = ((document.querySelector('meta[name="x-account-url"]')||{}).content||'').trim();
  if (meta) {                             // (2) use meta only if same-origin; otherwise downgrade
    try {
      var m = new URL(meta, location.href);
      if (m.origin === location.origin) return m.href;
    } catch(_e) {}
  }
  return location.origin + '/accounts.json';  // (3) same-origin fallback
})();
```

**Add meta to the main app head** (allows local override):

```html
<meta name="x-account-url" content="https://your.app.com/accounts.json">
```

**Drop-in login function** (top of main app):

```js
async function tryLogin(username, password) {
  try {
    const r = await fetch(ACCOUNT_URL, {cache: 'no-store'});
    const {salt, accounts} = await r.json();
    const account = accounts.find(a => a.u === username);
    if (!account) return {ok: false, err: 'Account not found'};
    const h = await hashPassword(salt, password);
    if (h !== account.h.toLowerCase()) return {ok: false, err: 'Wrong password'};
    localStorage.setItem('<?APP_KEY?>_unlocked', JSON.stringify({u: username, ts: Date.now()}));
    return {ok: true, u: username};
  } catch (e) {
    return {ok: false, err: 'Cannot reach account service ('+e.message+')'};
  }
}
```

**Verification:** open main app in Playwright; with correct credentials, `localStorage[<?APP_KEY?>_unlocked]` should have a value; with wrong password, the DOM should show "Wrong password".

## 3. Cross-page / cross-tab live config sync

**State:** when admin changes price / WeChat ID, every other open tab of the main app should see it within ~1s, no manual refresh.

**Approach:** double-trigger ——
1. **BroadcastChannel** (modern): admin writes `localStorage` and `postMessage({type:'cfg-update', ...})` at the same time.
2. **`storage` event** (fallback): admin's `setItem` automatically fires `storage` events in **other** tabs (same tab does NOT fire its own event).

```js
// admin side — "write" + broadcast
function publishCfg(cfg) {
  localStorage.setItem('<?APP_KEY?>_cfg', JSON.stringify(cfg));
  try {
    if (typeof BroadcastChannel !== 'undefined') {
      new BroadcastChannel('<?APP_KEY?>_admin').postMessage({type:'cfg-update', ...cfg});
    }
  } catch(_e) {}
}

// main app side — "subscribe" (register both channels)
function subscribeCfg(onChange) {
  // first load — read existing
  const saved = (() => { try { return JSON.parse(localStorage.getItem('<?APP_KEY?>_cfg')||'null'); } catch(_e){ return null; }})();
  if (saved) onChange(saved);
  // double-subscribe
  try {
    const bc = new BroadcastChannel('<?APP_KEY?>_admin');
    bc.onmessage = e => onChange(e.data);
  } catch(_e) {}
  window.addEventListener('storage', e => {
    if (e.key === '<?APP_KEY?>_cfg' && e.newValue) {
      try { onChange(JSON.parse(e.newValue)); } catch(_e) {}
    }
  });
}
```

**Verification:** open two tabs (tab1=admin, tab2=main app with the paywall mounted). In tab1, change price `19.9 → 29.9`. tab2 should receive the `cfg-update` message in < 500ms and update the UI in < 3s.

## 4. Paywall / member gate (optional)

**State:** once login is sorted, the main app must decide "which features are unlocked for logged-in users". Two common shapes:
1. **Soft lock** — when not logged in, show a teaser card; clicking it triggers the login popup; once logged in the card disappears and the content is shown.
2. **Hard gate** — when not logged in, replace the entire page with "Please log in"; once logged in, `location.reload()` to swap back.

**Approach:** tightly coupled to login state. A 3-state machine (`unknown` / `unlocked` / `locked`) inferred from `localStorage[<?APP_KEY?>_unlocked]` at startup:

```js
const PRO = { state: 'unknown', user: null };
(function bootstrapPro() {
  const saved = (() => { try { return JSON.parse(localStorage.getItem('<?APP_KEY?>_unlocked')||'null'); } catch(_e){ return null; }})();
  if (saved && saved.u && saved.ts) {
    PRO.state = 'unlocked'; PRO.user = saved.u;
  } else {
    PRO.state = 'locked';
  }
  // Feature modules render based on PRO.state (hard gate: `if (PRO.state!=='unlocked') showPaywall();`)
})();
```

Membership expiry — add `"paidUntil": "2026-12-31"` to an entry in `accounts.json`, validate at login:

```js
if (account.paidUntil && new Date(account.paidUntil) < new Date()) {
  return {ok: false, err: 'Membership expired. Please contact admin to renew.'};
}
```

**Verification:** with no login → paywall is shown; after login → paywall disappears; refresh → paywall still gone (persistence holds).

## 5. Optional backend: Node Functions for login / device binding

**State:** if you need "1 device per account" / "admin can remotely kick" / "last-login audit" — pure frontend can't do this. **This is the escape hatch:** a separate Node Functions endpoint + KV store, with an extra "Devices" tab in admin HTML.

**When to use it:**

- ✅ You want to tell customers "your account can't be shared by 5 people"
- ✅ You want to revoke a stolen account remotely
- ✅ You want to know "how many times did Alice log in last week"

**When to skip it:**

- ❌ You don't care about any of the above — pure-frontend hash saves ~90% of the maintenance cost vs Node Functions
- ❌ You've never deployed an EdgeOne Pages / Render Web Service — get the basics right first

**Minimum backend surface** (PoemGraph has this in `/Users/Zhuanz/Claude/PoemGraph/node-functions/`):

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/api/login` | POST | None (public, but needs username+password) | Main app calls; validates sha256 + checks device quota |
| `/api/admin/list` | POST | `Authorization: Bearer ${config:adminToken}` | Admin fetches all accounts' device binding state |
| `/api/admin/reset` | POST | Same | Clear a given account's device list |
| `/api/admin/add-device` | POST | Same | Manually attach a device UUID to an account |

KV key pattern: `account:<username>` → `{u, h, devices: [uuid1, uuid2], lastSeen}`.

> See Integration Checklist step 7.

## Pitfalls (already hit, captured here — don't hit them again)

1. **Salt duplicated across files → hash mismatch** — writing the `salt` constant in both admin HTML and the main app = guaranteed bug. **Convention:** salt appears **only** in `accounts.json` (filled in by admin when adding accounts); admin HTML and main app **read** its `salt` field, never hardcode any value. If you already have two copies, grep `salt` and unify to "read from JSON".

2. **`localStorage.unlocked=true` is not real auth** — anyone with browser dev tools can flip it. **Acknowledge it:** admin UI banner reads "**Zero-trust client, soft gate only**"; main app README reads "**This is a paywall, not auth**". Don't lie to yourself.

3. **Cross-origin `accounts.json` CORS trap** — Render/CF Pages default to same-origin and Just Work, but GitHub Pages + custom domain + cross-subdomain deployments fail silently at fetch. **Defense:** either keep admin and main app under one origin, or set `Access-Control-Allow-Origin: *` for `accounts.json` (caveat: exposes user existence). The easiest is one origin, zero CORS, zero fuss.

4. **`file://` is half-broken** — most browsers refuse WebCrypto on `file://` (`crypto.subtle === undefined`). Admin HTML must detect protocol:
   ```js
   if (location.protocol === 'file:') {
     document.body.insertAdjacentHTML('afterbegin',
       '<div class="warn">Serve via http://localhost; double-clicking file:// will not work.</div>');
   }
   ```
   **Always** tell users explicitly "run `python -m http.server` locally" instead of letting them poke around blind.

5. **Plaintext password leaking into localStorage** — convenient for debugging, catastrophic if shipped. **Forbidden:** localStorage holds `{u, ts}` + `paidUntil` only; reset form fields immediately after submit. Add a CI lint: `grep -rE 'localStorage.setItem.*\\bpwd|password' src/` must return empty.

6. **IndexedDB data vs new account id conflicts** (relevant for WhiteBoard-class projects) — `unlocked=true` should not change the ownership of existing IndexedDB records; re-check role on every write, **do not depend on cached login state**. Example: an IndexedDB whiteboard doc's ownership should follow URL params, not login state; login state only gates "can write".

7. **"User wants real auth, you delivered toy auth" expectation mismatch** — write "this is not auth, it's a paywall" in the README from day one, or you'll have a customer escalation later. **The only moment you can claim "real auth":** you ship Section 5's Node Functions + signed JWTs + server-side revocation. Until then, don't use the word "auth".

8. **`accounts.json` with placeholder accounts gets committed** — `accounts.json.example` goes into git, real `accounts.json` must be `.gitignore`'d. Otherwise the demo password's hash you stubbed becomes a real-world weak password. **Convention:** two files — example public, real gitignored, admin UI fetches the real one (same-host deployment), example exists only as a README reference for new clones.

## Integration checklist

> 8 steps for PoemGraph / WhiteBoard / any pure HTML project, in order.

1. **Copy the admin** — copy `templates/account-admin.html` to your project root and replace all 12 `<?...?>` placeholders with actual values (the most critical one is `<?APP_KEY?>`, which determines LS keys and the BroadcastChannel name).
2. **Prepare `accounts.json.example`** — commit it to the repo root; the UI defaults to `fetch ${origin}/accounts.json`; **the real `accounts.json` must be `.gitignore`'d** to avoid committing placeholder credentials.
3. **Add 3 meta tags to your main app's head**:
   ```html
   <meta name="x-account-url" content="./accounts.json">
   <meta name="x-<?APP_KEY?>-price" content="<?DEFAULT_PRICE?>">
   <meta name="x-<?APP_KEY?>-wx" content="<?OWNER_WECHAT?>">
   ```
4. **Add login + PRO bootstrap to the top of main app** — copy the drop-ins from Section 2, wire them to your login form's submit handler.
5. **Wire BroadcastChannel + storage subscription** — copy the drop-in from Section 3; verify across 2 tabs that admin price changes propagate in < 500ms.
6. **(Optional) Paywall** — Section 4's 3-state machine; pick soft-lock (card replaces content) or hard-gate (whole page replaces) per your needs.
7. **(Optional) Node Functions endpoints** — only if you need remote kick / device binding / audit; PoemGraph ships `/api/login + /api/admin/{list,reset,add-device}` already — copy `node-functions/` to your project root and follow EdgeOne Pages / Render docs to deploy.
8. **Deploy + CORS** — your host's `_headers` must serve `accounts.json` with `Content-Type: application/json`; if admin HTML and main app live on different subdomains, add `Access-Control-Allow-Origin: *` (caveat: exposes user-data; weigh the trade-off).

## Changelog

| Date | Change |
|---|---|
| 2026-07-13 | v1.0.0: extracted the PoemGraph `account-admin.html` pattern into a reusable skill. 4 files (SKILL.md / INTEGRATION.md / templates/account-admin.html / templates/accounts.json.example). Covers storage / login / cross-tab sync / paywall / optional-remote — five sections, 8 pitfalls, 8-step checklist. |
