# html-account-admin / 静态 HTML 零后端账号管理

[English](#english) · [中文](#中文)

> Purpose: explain what html-account-admin does, when to use it, and how to install it. Target readers: developers and indie creators adding accounts, member pages, or a lightweight paywall to static HTML projects. How to read: choose a language above, scan the honesty notice, then follow the install/integration steps.

---

## English

![Put index.html and accounts.json into a static site](./assets/readme/static-site-files.png)

`html-account-admin` is an account-management pattern for **pure static HTML projects**: one `account-admin.html` admin tool plus one repo-root `accounts.json` file. It works on Render, Cloudflare Pages, Vercel, GitHub Pages, EdgeOne Pages, and similar static hosts.

The point is simple: **you do not need to run your own backend**.

### What It Does

![Login and paid content without a backend](./assets/readme/no-server-paywall.png)

Use it when you want to:

- Add member login to a single-file HTML tool
- Add teaser/member-only content to a static site
- Add a lightweight paywall to a course, resource library, game, or knowledge graph
- Generate accounts locally with an admin page, then deploy `accounts.json` through git
- Validate a commercial idea before maintaining a database, server, or identity provider

Core features:

- **Account storage**: accounts live in `accounts.json` and deploy with the static site
- **Login check**: the main page fetches `accounts.json` and verifies password hashes in the browser
- **Member-state sync**: `BroadcastChannel + localStorage` keeps multiple tabs in sync
- **Admin tool**: a single-file `account-admin.html` with batch, auto-generate, verify, and parse flows
- **Static-host friendly**: works well with Render Static Site, Cloudflare Pages, Vercel, GitHub Pages, and EdgeOne Pages

### Important Honesty Notice

![From teaser to member content](./assets/readme/teaser-to-member.png)

**This is a paywall, not real authentication.**

Anyone who knows browser dev tools may bypass it by changing `localStorage` or reading front-end code. Use it to keep casual visitors away from paid content. Do not use it to protect secrets, private data, financial data, or enterprise permissions.

Do not use this pattern if:

- You already have a real backend
- You need OIDC / SSO / Auth0 / Supabase / Clerk / Firebase
- You need audit logs, server-side revocation, or strict device control
- You need to protect content that must never be exposed

Those cases need a standard backend auth system or a professional identity provider.

### How It Works

![Member login flow](./assets/readme/member-login-flow.png)

The basic flow:

1. The admin opens `account-admin.html`
2. The admin generates account/password pairs
3. The admin exports or copies `accounts.json`
4. `accounts.json` is placed at the static site root
5. The main HTML page loads that account file
6. The user enters username and password
7. The browser re-hashes and verifies the password locally
8. Member content is unlocked after a successful check

A typical project looks like this:

```text
my-static-site/
├── index.html
├── account-admin.html
├── accounts.json
└── assets/
```

### Admin Tool

![Batch account admin tool](./assets/readme/admin-batch-tool.png)

`templates/account-admin.html` is a single-file admin page you can copy into the target project. It includes these tabs:

- **Batch**: paste a batch of accounts and generate JSON
- **Auto**: generate account/password pairs automatically
- **Verify**: validate an account JSON file
- **Parse**: parse plaintext account lists and re-hash passwords

It is meant for the site owner/admin, not for public visitors.

### Hosting Options

![Deploy to multiple static hosting platforms](./assets/readme/static-hosting-options.png)

This pattern works on any host that can serve static files:

- Render Static Site
- Cloudflare Pages
- Vercel
- GitHub Pages
- Tencent EdgeOne Pages
- Netlify
- Any Nginx / Apache static directory

As long as your main page can read same-origin `accounts.json`, it can work.

### Invoke the Claude Code Skill

After installation, you can ask Claude Code things like:

```text
add login to my static HTML
paywall for static HTML
I need accounts on my single-file site
add a member login to PoemGraph/WhiteBoard
给这个静态 HTML 加账号
给 index.html 加 paywall
```

You can also call it explicitly:

```text
/html-account-admin add login to my static HTML
/html-account-admin 给 WhiteBoard 加账号管理
```

### Install

Copy the whole repository into Claude Code's user-level skills directory:

```bash
mkdir -p ~/.claude/skills/html-account-admin
cp -r ./html-account-admin/* ~/.claude/skills/html-account-admin/
ls ~/.claude/skills/html-account-admin/
```

You should see:

```text
SKILL.md
INTEGRATION.md
CHANGELOG.md
templates/
```

Make sure `SKILL.md` keeps this frontmatter flag:

```yaml
user-invocable: true
```

Otherwise `/html-account-admin` may not respond as an explicit command.

### Integrate Into Your Project

See [INTEGRATION.md](./INTEGRATION.md) for the full guide. Short version:

1. Copy `templates/account-admin.html` to the target project root
2. Create `accounts.json.example`
3. Add the real `accounts.json` to `.gitignore` if you keep local working copies
4. Add account config meta tags to your main HTML `<head>`
5. Add the login functions and PRO bootstrap near the top of the main HTML
6. Wire `BroadcastChannel + storage` synchronization
7. Choose soft-lock or hard-gate behavior for member content
8. Deploy and verify that the page can read `accounts.json`

---

## 中文

![把 index.html 和 accounts.json 放进静态站点](./assets/readme/static-site-files.png)

`html-account-admin` 是一个给 **纯静态 HTML 项目**使用的账号管理方案：一个 `account-admin.html` 管理工具，加一个仓库根目录的 `accounts.json`，就能在 Render / Cloudflare Pages / Vercel / GitHub Pages / EdgeOne Pages 等静态托管上做账号、会员入口和轻量付费墙。

它的重点是：**不需要自己起后端**。

### 这能做什么

![没有后端也能做登录和付费内容](./assets/readme/no-server-paywall.png)

适合这些场景：

- 给单文件 HTML 工具加「会员登录」
- 给静态站点加「试看 / 会员内容」
- 给课程、资料、小游戏、知识图谱等项目加轻量付费墙
- 用一个本地 admin 页面批量生成账号，再把 `accounts.json` 提交到仓库部署
- 需要快速验证商业想法，但暂时不想维护数据库、服务器或身份服务

核心能力：

- **账号存储**：账号列表放在 `accounts.json`，随静态站点一起部署
- **登录校验**：主页面 `fetch(accounts.json)` 后在浏览器端校验密码哈希
- **会员状态同步**：`BroadcastChannel + localStorage`，多个 tab 能实时同步
- **管理员工具**：单文件 `account-admin.html`，支持批量生成、自动生成、验证、解析
- **静态托管友好**：适合 Render Static Site、Cloudflare Pages、Vercel、GitHub Pages、EdgeOne Pages

### 重要诚实声明

![试看页到会员页](./assets/readme/teaser-to-member.png)

**这是 paywall，不是真正的安全认证系统。**

任何会用浏览器开发者工具的人，都可能通过改 `localStorage` 或读前端代码绕过它。所以它适合「让普通用户够不到会员内容」的软门槛，不适合保护机密数据、财务数据、隐私数据或企业权限。

不要在这些场景使用本方案：

- 你已经有真实后端
- 你需要 OIDC / SSO / Auth0 / Supabase / Clerk / Firebase
- 你需要审计日志、服务端吊销、严格设备控制
- 你要保护不能泄露的秘密内容

这些场景应该使用标准后端鉴权或专业身份服务。

### 它怎么工作

![会员登录流程](./assets/readme/member-login-flow.png)

基本流程是：

1. 管理员打开 `account-admin.html`
2. 批量生成账号和密码
3. 导出或复制 `accounts.json`
4. 把 `accounts.json` 放到静态站点根目录
5. 主 HTML 页面加载账号文件
6. 用户输入账号密码
7. 浏览器本地重算哈希并校验
8. 校验通过后解锁会员内容

项目文件通常长这样：

```text
my-static-site/
├── index.html
├── account-admin.html
├── accounts.json
└── assets/
```

### 管理员工具

![批量账号管理工具](./assets/readme/admin-batch-tool.png)

`templates/account-admin.html` 是一个可以直接复制到目标项目里的单文件管理页面，包含这些 tab：

- **Batch**：批量输入账号并生成 JSON
- **Auto**：自动生成一批账号密码
- **Verify**：检查账号 JSON 是否有效
- **Parse**：从明文账号列表解析并重算哈希

它适合作者自己使用，不建议直接暴露给普通用户。

### 部署平台

![可部署到多种静态托管平台](./assets/readme/static-hosting-options.png)

本方案适合任何能托管静态文件的平台：

- Render Static Site
- Cloudflare Pages
- Vercel
- GitHub Pages
- Tencent EdgeOne Pages
- Netlify
- 任意 Nginx / Apache 静态目录

只要你的主页面能读取同源的 `accounts.json`，就能工作。

### 如何触发 Claude Code skill

安装后，你可以这样对 Claude Code 说：

```text
给这个静态 HTML 加账号
给 index.html 加 paywall
我要账号/付费墙，不想起后端
给 PoemGraph/WhiteBoard 加会员登录
add login to my static HTML
paywall for static HTML
```

也可以手动调用：

```text
/html-account-admin 给 WhiteBoard 加账号管理
/html-account-admin help me add login to my static HTML
```

### 安装

把整个仓库复制到 Claude Code 的 user-level skills 目录：

```bash
mkdir -p ~/.claude/skills/html-account-admin
cp -r ./html-account-admin/* ~/.claude/skills/html-account-admin/
ls ~/.claude/skills/html-account-admin/
```

应该能看到：

```text
SKILL.md
INTEGRATION.md
CHANGELOG.md
templates/
```

确认 `SKILL.md` frontmatter 里保留了：

```yaml
user-invocable: true
```

否则 `/html-account-admin` 手动调用可能不会响应。

### 接入到你的项目

详细步骤见 [INTEGRATION.md](./INTEGRATION.md)。简版流程：

1. 复制 `templates/account-admin.html` 到目标项目根目录
2. 创建 `accounts.json.example`
3. 将真实 `accounts.json` 加入 `.gitignore`，避免误提交明文工作文件
4. 在主 HTML 的 `<head>` 加账号配置 meta
5. 在主 HTML 顶部加登录函数与 PRO bootstrap
6. 接入 `BroadcastChannel + storage` 同步
7. 选择软锁或硬门方式保护会员内容
8. 部署并检查 `accounts.json` 是否能被页面读取

---

## File Structure / 文件结构

```text
html-account-admin/
├── README.md
├── README.en.md
├── SKILL.md
├── INTEGRATION.md
├── CHANGELOG.md
├── assets/
│   └── readme/
│       ├── admin-batch-tool.png
│       ├── member-login-flow.png
│       ├── no-server-paywall.png
│       ├── static-hosting-options.png
│       ├── static-site-files.png
│       └── teaser-to-member.png
└── templates/
    ├── account-admin.html
    └── accounts.json.example
```

## GitHub README Tabs / 关于 GitHub README 的 Tab

GitHub Markdown does **not** support real tab controls inside README files. This README uses anchor links instead:

GitHub Markdown **不支持真正的 tab 切换控件**。这个 README 使用页内锚点模拟语言入口：

- [English](#english)
- [中文](#中文)

## License / 许可

MIT. You may copy, modify, and redistribute it. Please keep the PoemGraph attribution.

MIT。可以复制、修改、再发布；请保留 PoemGraph 来源说明。

## 变更记录

| 日期 | 变更内容 |
|------|---------|
| 2026-07-13 | 将根 README 调整为 GitHub 首页语言入口样式：顶部 `English · 中文` 锚点，正文按英文/中文两段独立展示，并保留 6 张漫画说明图。 |
| 2026-07-13 | 拆分中英文 README，并加入 6 张漫画说明图，让 GitHub 首页更清晰易读。 |
| 2026-07-13 | v1.0.0 初始创建：从 PoemGraph `account-admin.html` 抽出通用 skill。 |
