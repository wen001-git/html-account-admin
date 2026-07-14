# html-account-admin (English)

This is the English mirror of the README. The full English docs are:

- [`SKILL.md`](./SKILL.md) — the 5-section method + 8 pitfalls + 8-step checklist
- [`INTEGRATION.md`](./INTEGRATION.md) — 8-step integration guide + FAQ
- [`CHANGELOG.md`](./CHANGELOG.md) — skill's own change history

For the **top-level (bilingual) repo landing page** with installation, invocation, and FAQ, see [`../README.md`](../README.md).

For the templated admin HTML / accounts JSON schema, see [`./templates/`](./templates/).

---

## TL;DR for English readers

Add a "members only" section to your static HTML site without a backend. One admin HTML file + one JSON file, both committed to git, both deployed to your static host (Render / Cloudflare Pages / Vercel / GitHub Pages / EdgeOne Pages).

**This is a paywall, not real auth.** Anyone with browser dev tools can flip `localStorage.unlocked=true` and bypass login. Use it to hide paid content from casual visitors — never to protect secrets.

---

## Install (Claude Code)

Drop the top-level `html-account-admin/` folder into `~/.claude/skills/` and restart Claude Code. The skill auto-triggers when you ask for accounts / paywalls on a static HTML project, or call it explicitly with `/html-account-admin`.

See the top-level README for full trigger-mode documentation.
