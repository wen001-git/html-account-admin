# HTML Account Admin · Integration Guide

> **Purpose:** let any "single-file HTML + static hosting" project add the `html-account-admin` skill in 8 steps. **Audience:** project owners / integrators / AI coding tools implementing the integration. **How to read:** execute steps 1–8 in order; after each step, run its "self-check" before moving on.

## Prerequisites (verify before starting)

- Project shape: **single-file HTML + static hosting** (Render / CF Pages / Vercel / GitHub Pages / EdgeOne Pages).
- **No backend, and you don't want one** (a single Node Function is optional).
- The project has an "admin" role (yourself) — this skill is **not** for user self-signup.

## 8 steps

### Step 1: Copy the admin

Copy `templates/account-admin.html` from this skill into your project root. Rename it to `<your-app>-account-admin.html` (e.g. `whiteboard-account-admin.html`).

Replace every `<?...?>` placeholder with values for your project. **Required:**

| Placeholder | Example value | Used in |
|---|---|---|
| `<?APP_NAME?>` | `WhiteBoard Pro` | title, header, tab labels |
| `<?APP_KEY?>` | `whiteboard` | BroadcastChannel name, LS keys, meta defaults |
| `<?APP_DOMAIN_HINT?>` | `whiteboard.app.com` | "Open main app" button + help text |
| `<?SALT?>` | `wb-salt-v1-2026` (length ≥ 16, any unique string) | Hash computation |
| `<?DEFAULT_PRICE?>` | `$5` / `¥29.9` / etc. | Three price fields |
| `<?ADMIN_TABS?>` | 5-tab HTML (see below) | Tab navigation |

The remaining 6 (`<?FAVICON?>` / `<?OG_META?>` / etc.) can be edited or left as-is.

**Minimum `<?ADMIN_TABS?>` skeleton** (replacing the original 5 tabs):

```html
<div class="tabs">
  <button class="tab on" data-tab="auto">🎲 Auto-generate</button>
  <button class="tab" data-tab="batch">Batch generate</button>
  <button class="tab" data-tab="verify">Verify</button>
  <button class="tab" data-tab="parse">Parse</button>
</div>
```

**Self-check:** open `<your-app>-account-admin.html` in a browser; `document.title` should reflect the new name and the tab labels should match your product.

### Step 2: Prepare `accounts.json.example` + real `.gitignore`

Create two files at the repo root:

**`accounts.json.example`** (committed, README reference only):

```json
{
  "version": 1,
  "salt": "<?SALT?>",
  "accounts": []
}
```

**`accounts.json`** (the real one — **must** be `.gitignore`'d):

```bash
echo "accounts.json" >> .gitignore
```

> See [Pitfall #8](#pitfalls-already-hit-captured-here--dont-hit-them-again) (the big block above): accidentally committing `accounts.json` ships your stub demo-password hashes into production.

**Self-check:** `git status` must not show `accounts.json`; `accounts.json.example` must be visible.

### Step 3: Add 3 meta tags to the main app's `<head>`

Open your main app (your `index.html` or `poemgraph-pro.html`) and add inside `<head>`:

```html
<meta name="x-account-url" content="./accounts.json">
<meta name="x-<?APP_KEY?>-price" content="<?DEFAULT_PRICE?>">
<meta name="x-<?APP_KEY?>-wx" content="<?OWNER_WECHAT?>">
```

Substitute `<?APP_KEY?>` / `<?DEFAULT_PRICE?>` / `<?OWNER_WECHAT?>` with the same values from Step 1.

**Self-check:** browser inspector should show these three `<meta>` tags in the main app's head.

### Step 4: Add login + PRO bootstrap to the top of the main app

Copy the two drop-in functions (`hashPassword` + `tryLogin`) from [SKILL.md §2](./SKILL.md#2-html-side-login-fetch--rehash) into a `<script>` tag at the top of the main app.

Then add a minimal login UI (a form + status div wherever it makes sense):

```html
<form id="login-form">
  <input id="login-user" placeholder="Username">
  <input id="login-pass" type="password" placeholder="Password">
  <button type="submit">Sign in</button>
</form>
<div id="login-status"></div>
<script>
document.getElementById('login-form').onsubmit = async (e) => {
  e.preventDefault();
  const u = document.getElementById('login-user').value.trim();
  const p = document.getElementById('login-pass').value;
  const r = await tryLogin(u, p);
  document.getElementById('login-status').textContent = r.ok ? '✓ Logged in as ' + r.u : '❌ ' + r.err;
  if (r.ok) location.reload();
};
</script>
```

**Self-check:** in admin HTML, switch to the "Batch generate" tab → enter `demo:demo123` → click "Generate" → copy the JSON → save to `accounts.json`. Then verify in admin's "Verify" tab that `demo` + `demo123` → "match=true". Then open the main app in browser, enter `demo` / `demo123` → expect "✓ Logged in as demo".

### Step 5: BroadcastChannel + storage subscription

Copy `subscribeCfg` from [SKILL.md §3](./SKILL.md#3-cross-page--cross-tab-live-config-sync) into the main app. Wherever your paywall / price display lives, add:

```js
subscribeCfg(cfg => {
  document.querySelectorAll('[data-price]').forEach(el => el.textContent = cfg.price);
  document.querySelectorAll('[data-wx]').forEach(el => el.textContent = cfg.wx);
});
```

**Self-check:** open 2 tabs (tab1 = admin, tab2 = main app with paywall mounted). In tab1, change price `19.9 → 29.9`. tab2 should update the price text within < 1s.

### Step 6: (Optional) Paywall

See [SKILL.md §4](./SKILL.md#4-paywall--member-gate-optional) for the 3-state machine. Pick soft-lock (card replaces content) or hard-gate (whole page replaces) per your need. Minimal hard-gate:

```js
// append to PRO.bootstrap:
if (PRO.state !== 'unlocked') {
  document.body.innerHTML = `
    <div style="padding:60px 24px;text-align:center;max-width:480px;margin:0 auto">
      <h1>Please sign in</h1>
      <p>This content requires a paid account.</p>
      <!-- paste the login form from Step 4 here -->
    </div>
  `;
}
```

**Self-check:** without login → "Please sign in" shows; after login → paywall gone; refresh → paywall still gone (persistence holds).

### Step 7: (Optional) Node Functions endpoints

Only do this if you need **remote kick / device binding / last-login audit.** Copy these 6 files from PoemGraph's `node-functions/` directory into your project's same-named directory:

- `_auth.js`
- `_kv.js`
- `login.js`
- `admin-list.js`
- `admin-reset.js`
- `admin-add-device.js`

Deploy following EdgeOne Pages / Render Web Service docs. Admin HTML's "Devices" tab uses them as-is.

**Self-check:** after deploy, `curl -X POST https://your.app.com/api/admin/list -H "Authorization: Bearer YOUR_TOKEN"` should return `{maxDevices: N, accounts: [...]}`.

### Step 8: Deploy + CORS

Deploy your project to Render / Cloudflare Pages / Vercel. **Critical:** add a `_headers` file at the repo root (CF Pages / Netlify syntax):

```
/accounts.json
  Content-Type: application/json
```

If admin HTML and main app live on different subdomains, also add:

```
/accounts.json
  Access-Control-Allow-Origin: *
```

> Note: `*` exposes "user exists" to any site. When admin and main app share a subdomain you don't need this line.

**Self-check:** `curl -I https://your.app.com/accounts.json` returns 200 + `Content-Type: application/json`. Browser console `await fetch('/accounts.json').then(r=>r.json())` returns valid JSON.

---

## After integration — FAQ

### Q: A customer forgot their password — what do I do?

Open admin HTML → "Single hash" tab → enter the new password → generate hash → replace that account's `h` in `accounts.json` → `git push` → customer uses the new password. **No outbound message needed** — the old password simply stops working, and you DM the new one privately.

### Q: Should I obfuscate the JS?

**No.** This is a paywall, not auth; even if the JS literally says `if (password === 'demo123') unlock()`, all it does is move the "how to bypass" question from dev-tools localStorage to reading the source — there's no secret to hide.

### Q: One console line `localStorage[unlocked]=true` bypasses the whole thing, right?

Right. That's why this is a paywall, not auth (see SKILL.md TL;DR table footnote). If you really need to block "people who edit localStorage", wire Step 7's Node Functions in.

### Q: Can I add 2FA / MFA inside the admin HTML?

**Not recommended.** This skill has no email / SMS / OTP channel; bolting it on turns admin HTML into a half-baked "backend". If you need real MFA, use Auth0 / Supabase / Clerk (counter-example: this skill deliberately does NOT integrate those — see SKILL.md frontmatter reverse triggers).

---

## Related docs

- [SKILL.md](./SKILL.md) — the method itself (required reading)
- [CHANGELOG.md](./CHANGELOG.md) — this skill's own change history
- [templates/account-admin.html](./templates/account-admin.html) — 12-placeholder admin HTML template
- [templates/accounts.json.example](./templates/accounts.json.example) — schema example
- Reference implementation: [PoemGraph repo](https://github.com/your-repo/PoemGraph) — the live site where this pattern came from

## Changelog

| Date | Change |
|---|---|
| 2026-07-13 | v1.0.0: extracted from PoemGraph `account-admin.html` into a reusable skill; 8-step integration guide + FAQ + related-doc links. |
