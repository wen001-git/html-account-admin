# HTML Account Admin · 接入指南

> 目的：让任何「单文件 HTML + 静态托管」项目按 8 步接入 `html-account-admin` skill。　目标读者：项目所有者 / 实施接入的开发者 / 给项目接入的 AI 编程助手。　如何阅读：从第 1 步顺序执行；每步操作完都做那一步末尾的「自检」再进下一步。

## 前置条件（开始前确认）

- 项目形态：**单文件 HTML + 静态托管**（Render / CF Pages / Vercel / GitHub Pages / EdgeOne Pages）。
- **没有也不能加后端**（或愿起 1 个 Node Functions，可选）。
- 项目里有一个「作者/管理员」角色（自己）——本 skill 不面向「用户自助注册」。

## 8 步走

### 第 1 步：复制 admin

把 skill 自带的 `templates/account-admin.html` 拷到你的项目根，命名为 `<your-app>-account-admin.html`（如 `whiteboard-account-admin.html`）。

把所有 `<?...?>` 占位符替换成你的项目值。**必改**：

| 占位符 | 示例替换值 | 用在哪 |
|---|---|---|
| `<?APP_NAME?>` | `WhiteBoard Pro` | title、header、tab 标签 |
| `<?APP_KEY?>` | `whiteboard` | BroadcastChannel 名、LS key、meta 默认 |
| `<?APP_DOMAIN_HINT?>` | `whiteboard.app.com` | 「打开主 app」按钮 + help text |
| `<?SALT?>` | `wb-salt-v1-2026`（长度 ≥ 16，自定义任意字符串） | hash 计算 |
| `<?DEFAULT_PRICE?>` | `¥29.9` / `$5` 等 | 三条价格字段 |
| `<?ADMIN_TABS?>` | 5 个 tab 的 HTML（见下） | tab 导航 |

剩下的 6 个（`<?FAVICON?>` / `<?OG_META?>` 等）按需改或不改。

**`<?ADMIN_TABS?>` 的最小骨架**（替换原 5 tab）：

```html
<div class="tabs">
  <button class="tab on" data-tab="auto">🎲 自动生成</button>
  <button class="tab" data-tab="batch">批量生成</button>
  <button class="tab" data-tab="verify">验证</button>
  <button class="tab" data-tab="parse">解析</button>
</div>
```

**自检**：浏览器打开 `<your-app>-account-admin.html`，确认 `document.title` 显示新名字、tab 标签也变成「白板」相关内容。

### 第 2 步：准备 `accounts.json.example` + 真 `.gitignore`

仓库根创建两个文件：

**`accounts.json.example`**（提交到 git，仅作 README 参考）：

```json
{
  "version": 1,
  "salt": "<?SALT?>",
  "accounts": []
}
```

**`accounts.json`**（真实文件，**必须** `.gitignore`）：

```bash
echo "accounts.json" >> .gitignore
```

> 见 [踩过的坑 #8](#踩坑-8-accountsjson-写-placeholder-被-commit)：真实 `accounts.json` 误提交等于直接在线泄露你 stub 写的弱口令 hash。

**自检**：`git status` 不能看到 `accounts.json`；`accounts.json.example` 必须能被看到。

### 第 3 步：主 app HTML 头部加 3 个 meta

打开主 app（你的 `index.html` 或 `poemgraph-pro.html`），在 `<head>` 段加：

```html
<meta name="x-account-url" content="./accounts.json">
<meta name="x-<?APP_KEY?>-price" content="<?DEFAULT_PRICE?>">
<meta name="x-<?APP_KEY?>-wx" content="<?OWNER_WECHAT?>">
```

把 `<?APP_KEY?>` / `<?DEFAULT_PRICE?>` / `<?OWNER_WECHAT?>` 替换成第 1 步的同一组值。

**自检**：浏览器 inspect 主 app head，应能看到这 3 个 `<meta>`。

### 第 4 步：主 app 顶部加 login + PRO bootstrap

把 [SKILL.md 第二节](./SKILL.md#二html-端登录校验fetch--重哈希) 的 2 个 drop-in 函数（`hashPassword` + `tryLogin`）粘到主 app `<head>` 里 `<script>` 节点。

再加一个登录 UI：简单 1 个表单 + 1 个 status `<div>`（位置选你合适的位置）：

```html
<form id="login-form">
  <input id="login-user" placeholder="用户名">
  <input id="login-pass" type="password" placeholder="密码">
  <button type="submit">登录</button>
</form>
<div id="login-status"></div>
<script>
document.getElementById('login-form').onsubmit = async (e) => {
  e.preventDefault();
  const u = document.getElementById('login-user').value.trim();
  const p = document.getElementById('login-pass').value;
  const r = await tryLogin(u, p);
  document.getElementById('login-status').textContent = r.ok ? '✓ 已登录 ' + r.u : '❌ ' + r.err;
  if (r.ok) location.reload();
};
</script>
```

**自检**：admin HTML「批量生成」tab → 输入 `demo:demo123` → 「生成」→ 复制 JSON → 保存到 `accounts.json`；在 admin 的「验证」tab 验证 `demo` + `demo123` → 应显示「匹配=true」。然后浏览器打开主 app，输入 demo / demo123 → 应显示「✓ 已登录 demo」。

### 第 5 步：BroadcastChannel + storage 订阅

把 [SKILL.md 第三节](./SKILL.md#三跨页跨-tab-实时配置同步) 的 `subscribeCfg` 粘到主 app。在你的付费墙 / 价格展示位置加：

```js
subscribeCfg(cfg => {
  document.querySelectorAll('[data-price]').forEach(el => el.textContent = cfg.price);
  document.querySelectorAll('[data-wx]').forEach(el => el.textContent = cfg.wx);
});
```

**自检**：开 2 个 tab（tab1=admin，tab2=主 app 含付费墙），在 tab1 改价格 `19.9 → 29.9`；tab2 应在 < 1s 内更新价格文字。

### 第 6 步：(可选)付费墙

参考 [SKILL.md 第四节](./SKILL.md#四付费墙会员门控可选) 的 3 状态机，按你页面需要选软锁（卡片替换）还是硬门（整页替换）。最简单的硬门示例：

```js
// 在 PRO.bootstrap 末尾加：
if (PRO.state !== 'unlocked') {
  document.body.innerHTML = `
    <div style="padding:60px 24px;text-align:center;max-width:480px;margin:0 auto">
      <h1>请先登录</h1>
      <p>这是付费内容，登录后查看。</p>
      <!-- 把第 4 步的登录表单塞这里 -->
    </div>
  `;
}
```

**自检**：未登录打开主 app → 应看到「请先登录」；登录后 → 看不到付费墙；刷新 → 仍能看到付费内容（持久化生效）。

### 第 7 步：(可选)Node Functions 端点

仅当你需要**远程踢人/设备绑定/最后登录审计**时做这件事：

把 `/Users/Zhuanz/Claude/PoemGraph/node-functions/` 目录下的 `_auth.js` / `_kv.js` / `login.js` / `admin-list.js` / `admin-reset.js` / `admin-add-device.js` 6 个文件拷到你的项目同名目录，按 EdgeOne Pages / Render Web Service 文档部署。

`admin.html` 的「设备管理」tab 直接用，无需改。

**自检**：部署后 `curl -X POST https://your.app.com/api/admin/list -H "Authorization: Bearer YOUR_TOKEN"` 应返回 `{maxDevices: N, accounts: [...]}`。

### 第 8 步：部署 + CORS

把项目部署到 Render / Cloudflare Pages / Vercel。**关键**：仓库根加 `_headers` 文件（CF Pages）或 Netlify `_headers`：

```
/accounts.json
  Content-Type: application/json
```

如果 admin HTML 与主 app 在不同子域，额外加：

```
/accounts.json
  Access-Control-Allow-Origin: *
```

> 注意：`*` 会把"账号存在性"暴露给任何网站。admin 与主 app 在同子域时**不需要**这行。

**自检**：`curl -I https://your.app.com/accounts.json` 应返回 200 + `Content-Type: application/json`；浏览器 console 跑 `await fetch('/accounts.json').then(r=>r.json())` 应能拿到 JSON。

---

## 接入后常见问题

### Q：用户忘了密码怎么办？

admin HTML「单条哈希」tab → 输入新密码生成新 hash → 替换 `accounts.json` 里那个账号的 `h` 字段 → `git push` → 用户用新密码登录。**无需**给用户发任何东西（旧密码作废，新密码由你私聊发送）。

### Q：要不要给 JS 加混淆？

**不用**。这是 paywall 不是 auth；JS 里就算明文写 `if (password === 'demo123') unlock()` 也只是把"绕过复杂度"从 dev tools 改 localStorage 简化成读源码——没有任何需要隐藏的逻辑。

### Q：浏览器 console 一行 `localStorage[unlocked]=true` 就能绕过，对吗？

对。所以这是 paywall，不是 auth（见 SKILL.md「先说结论」表格最后一行）。如果你真的需要"挡住会改 localStorage 的人"，请接第 7 步的 Node Functions。

### Q：admin HTML 我想加上 2FA/MFA 行吗？

**不建议**。本 skill 不接邮件/短信/任何 OTP 通道；强行加只会让 admin HTML 长成一只臃肿的"半成品后端"。真要 MFA 请走 Auth0 / Supabase / Clerk（反面教材：本 skill 也不接这些——见 SKILL.md frontmatter 反 trigger）。

---

## 关联文档

- [SKILL.md](./SKILL.md) — 方法本体（必读）
- [CHANGELOG.md](./CHANGELOG.md) — skill 自身的变更历史
- [templates/account-admin.html](./templates/account-admin.html) — 12 占位符的 admin HTML 模板
- [templates/accounts.json.example](./templates/accounts.json.example) — schema 示例
- 反例参考：[PoemGraph 实施版本](https://github.com/your-repo/PoemGraph) — 同 skill 但已落地到 PoemGraph 项目里的具体形态

## 变更记录

| 日期 | 变更 |
|---|---|
| 2026-07-13 | v1.0.0：从 PoemGraph `account-admin.html` 抽出通用 skill，8 步入接指南 + Q&A 段 + 关联文档链接。 |
