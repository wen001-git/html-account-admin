# html-account-admin · 给纯 HTML 项目的零后端账号管理 / Zero-Backend Account Admin for Static HTML

> **目的**：把 PoemGraph 的「单文件 admin HTML + 仓库根 `accounts.json` + WebCrypto」模式抽出来，做成 Claude Code 可自动触发的可复用 skill。  
> **Purpose**: extract the PoemGraph "single-file admin HTML + repo-root accounts.json + WebCrypto" pattern into a reusable Claude Code skill that auto-triggers when you ask for accounts / paywalls on a static HTML project.

---

## 这能做什么 / What this does

把任意「**单文件 HTML + 静态托管**」项目（如 Render / Cloudflare Pages / Vercel / GitHub Pages）在 **不起后端** 的前提下，加上一套可用的账号 / 付费墙 / 会员状态管理：

- **账号存储** — `accounts.json` 提交到仓库根，`git push` 即部署
- **登录校验** — 主 app `fetch + 重哈希`，无需 JWT
- **跨 tab 实时同步** — `BroadcastChannel + localStorage` 双保险
- **管理员工具** — 单文件 admin HTML，含批量 / 自动生成 / 验证 / 解析 4 个 tab

Add a working **account + paywall + member gate** to any **single-file HTML + static hosting** project (Render / Cloudflare Pages / Vercel / GitHub Pages) — **without a backend**:

- **Storage** — `accounts.json` lives in the repo root; `git push` = deploy
- **Login** — main app `fetch + rehash`, no JWT
- **Cross-tab sync** — `BroadcastChannel + localStorage` as double-trigger
- **Admin tool** — single-file `account-admin.html` with batch / auto-generate / verify / parse tabs

---

## ⚠️ 重要诚实声明 / Important honesty notice

**这是 paywall，不是 auth。** [`./SKILL.md`](./SKILL.md) 顶部的正面 trigger 后面紧跟 5 类**反面 trigger**：已经有真后端、想用 OIDC、想接 Auth0/Supabase/Clerk/Firebase、需要审计、需要服务端吊销 — 那些场景**别用本 skill**，请走标准身份方案。任何持有浏览器 dev tools 的人都能改 `localStorage.unlocked=true` 绕过登录；本 skill 只适合「让普通人够不到付费内容」的软门场景。

**This is a paywall, NOT auth.** The 5 reverse triggers in [`SKILL.md`](./SKILL.md) frontmatter (existing backend / OIDC / Auth0·Supabase·Clerk·Firebase / audit / server-side revocation) are explicit DO-NOT-USE cases — use a real identity provider instead. Anyone with browser dev tools can flip `localStorage.unlocked=true` to bypass login; this skill is for "make it inconvenient for ordinary users" only, never "block attackers".

---

## 怎么触发本 skill / How to invoke

装上之后，**三种触发方式**任选其一：

Once installed, **three invocation modes** — pick whichever fits:

### 方式 1：自动触发（推荐 · 大多数场景） / Auto-trigger (recommended · most cases)

Claude Code 看到你的消息**含正面 trigger 关键词**时会自动加载本 skill。下面这些说法都能触发：

> "给这个静态 HTML 加账号" / "我要账号/付费墙" / "给 PoemGraph/WhiteBoard 加会员登录" /
> "我不想起后端但要给单文件 HTML 加 auth" / "给 index.html 加 paywall" /
> "add login to my static HTML" / "I need accounts on my single-file site" /
> "paywall for static HTML" / "membership gate without backend"

**反面 trigger 不会被加载**（你的消息含这些的话 Claude 不会用本 skill，会去找真鉴权方案）：

> 别用本 skill："OIDC / SSO / Auth0 / Supabase / Clerk / Firebase" + "audit / 审计 / server-side revocation"

### 方式 2：手动 `/<skill>` 调用（确定要用的场合） / Manual `/<skill>` invocation

如果你**确定要用本 skill**，在 Claude Code prompt 里直接打（注意：`user-invocable: true` 必须在 SKILL.md 的 frontmatter 里——本 skill 已开）：

```
/html-account-admin 给 WhiteBoard 加账号管理
/html-account-admin help me add login to my static HTML
```

> **Enablement check**: this only works if `SKILL.md` has `user-invocable: true` in its frontmatter. The shipped `SKILL.md` does — if you fork and edit, keep this flag.

### 方式 3：在另一个 AI 工具里手挂 / Reference by URL

非 Claude Code 工具（Cline / Continue / Cursor / Codex）不能自动加载 skill。两种用法：

1. **手动 cat 给 AI 看**：
   ```bash
   cat SKILL.md
   ```
   把完整文件粘进那个工具的上下文。
2. **放 GitHub Pages，加 URL 到工具 context**：
   推完后用 `raw.githubusercontent.com/<you>/html-account-admin/main/SKILL.md` 这种 URL，工具自己 fetch。

### Trigger 模式速查表 / Trigger cheat-sheet

| 你想要的 | 怎么说 | 走哪种 trigger |
|---|---|---|
| 给现成静态 HTML 加账号 | 「给 WhiteBoard 加账号」 | 自动 |
| 给新项目从零搭账号体系 | 「我要做一个 HTML 工具，需要账号系统，不要后端」 | 自动 |
| 在白板做付费内容 | 「白板加 paywall」 | 自动 |
| 改现有的 PoemGraph admin | 「PoemGraph 的 admin 我想加批量编辑」 | 自动 |
| **真鉴权 / OIDC / Auth0** | 「我要接 Auth0」 | **不会**触发本 skill，会去找真方案 |
| 已经确定用本 skill | `/html-account-admin` | 手动 |

---

## 装上 / Install

把整个 `html-account-admin/` 文件夹拷到 `~/.claude/skills/`，重启 Claude Code。之后任何对话里提到「给我这个静态 HTML 加账号管理」就会自动触发本 skill。

```bash
# 拷到 Claude Code 的 user-level skills 目录
mkdir -p ~/.claude/skills/html-account-admin
cp -r ./html-account-admin/* ~/.claude/skills/html-account-admin/
ls ~/.claude/skills/html-account-admin/   # 应该看到 SKILL.md / INTEGRATION.md / CHANGELOG.md / templates/
```

Drop the entire folder into `~/.claude/skills/` and restart Claude Code. The skill auto-triggers when you ask for accounts / paywalls on a static HTML project.

**验证装上了** / **Verify it's installed**:

```bash
# 应该能 grep 到 SKILL.md 的 frontmatter
head -3 ~/.claude/skills/html-account-admin/SKILL.md
# → 第一行应是 "---"
# → 应含 "name: html-account-admin"
# → 应含 "user-invocable: true"（手动 / 命令的前置条件）
```

如果装了但 `/html-account-admin` 不响应 — 大概率是 `user-invocable: true` 被误删了，查 `head -10 SKILL.md` 看到就 OK。

---

## 在你的项目里用 / Use in your project

按 [INTEGRATION.md](./INTEGRATION.md) 的 8 步顺序执行（每步末尾有 self-check）：

1. 复制 `templates/account-admin.html` 到项目根
2. 创建 `accounts.json.example`，把真实 `accounts.json` 加进 `.gitignore`
3. 主 app `<head>` 加 3 个 meta
4. 主 app 顶部加 login 函数 + PRO bootstrap
5. 接 BroadcastChannel + storage 订阅
6. （可选）付费墙 — 软锁 / 硬门二选一
7. （可选）Node Functions 端点 — 仅当你需要远程踢人 / 设备绑定 / 审计
8. 部署 + CORS

Follow the 8-step [INTEGRATION.md](./INTEGRATION.md) (each step ends with a self-check):

1. Copy `templates/account-admin.html` to your project root
2. Create `accounts.json.example`; add the real `accounts.json` to `.gitignore`
3. Add 3 meta tags to your main app's `<head>`
4. Add login function + PRO bootstrap to top of main app
5. Wire BroadcastChannel + storage subscription
6. (Optional) Paywall — soft-lock or hard-gate
7. (Optional) Node Functions endpoints — only if you need remote kick / device binding / audit
8. Deploy + CORS

---

## 文件结构 / File structure

```
html-account-admin/
├── README.md                       ← 你正在看的（含触发方式 + 安装 + 接入）
├── SKILL.md                        ← 5 节方法 + 8 踩坑 + 8 接入清单
├── INTEGRATION.md                  ← 8 步入接指南 + Q&A
├── CHANGELOG.md                    ← v1.0.0 初始创建
└── templates/
    ├── account-admin.html          ← 12 个 <?...?> 占位符的 admin HTML 模板
    └── accounts.json.example       ← schema 模板
```

---

## 谁在用 / Who's using it

- **[PoemGraph](https://github.com/WenLee/poemgraph)** — 诗词知识图谱，PoC 来源项目；本 skill 的核心机制就是从这里抽出来的。/ source project this skill was extracted from.

---

## 许可 / License

MIT — 复制、修改、再发布都行；保留 `PoemGraph` 出处注释即可。

MIT — copy / modify / redistribute freely; keep the PoemGraph attribution.

---

## 变更记录 / Changelog

| 日期 / Date | 变更 / Change |
|---|---|
| 2026-07-13 | v1.0.0 初始创建 / Initial release：从 PoemGraph `account-admin.html` 抽出通用 skill。 |
