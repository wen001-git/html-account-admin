---
name: html-account-admin
user-invocable: true
description: 当一个纯前端 HTML 项目（单文件 .html + 静态托管）需要加账号/付费墙/会员状态、又不想起后端、不要 JWT 时使用；典型场景像 PoemGraph 那样把 {u, sha256(salt+password)} 存进仓库根的 accounts.json，前端 fetch + 重哈希登录，跨 tab 用 BroadcastChannel + localStorage 同步。**不要在以下场景使用**：已经有真实后端 / 想用 OIDC SSO / 想接 Auth0 Supabase Clerk Firebase / 用户量超过个位数需要审计 / 任何需要服务器端吊销 token 的场景——那种情况请走标准身份方案，这条 skill 只给「零后端的玩具级会员门」，不替代真鉴权。
---

# 给纯 HTML 项目做零后端账号管理

> 目的：把 PoemGraph `account-admin.html` 这套「零后端、纯前端 HTML 账号管理」方法封装成通用 skill，让任何单文件静态 HTML 项目（PoemGraph、WhiteBoard 及其他）能按需复用。　目标读者：(1) 想给单文件 HTML 加账号/付费墙/会员门的开发者；(2) 给项目接入这套方案的 AI 编程助手。　如何阅读：先看「先说结论」确认价值，再看「五步方法」落地，最后看「接入检查清单」对照执行。

## 先说结论

这条 skill 解决一个非常窄的问题：**「我只有静态托管（Render / Cloudflare Pages / GitHub Pages），想给单文件 HTML 加账号/付费墙/会员门，又绝不打算起后端」**。方法核心是 WebCrypto + 文件存储——浏览器侧 `sha256(salt + password)` 把账号哈希列表存进仓库根的 `accounts.json`，主 app `fetch` 下来再重哈希校验登录；跨页/跨 tab 实时配置用 `BroadcastChannel + localStorage` 双保险。

| 维度 | 本 skill（PoemGraph 路线） | 真后端 auth 路线 |
|---|---|---|
| 需要服务端 | **不需要**（仅静态托管即可） | 必须（数据库 + API + TLS） |
| 密码存储 | `sha256(salt + password)` 单向哈希 | bcrypt/argon2 + 服务器侧 |
| 会话/token | `localStorage.unlocked=true`（dev tools 可改，**故意不强**） | JWT/cookie + 服务器侧吊销 |
| 多设备管理 | 无（任意设备登录后即等同于"已授权"） | 可强制每账号 N 台 |
| 审计/日志 | 无 | 服务器侧完整记录 |
| 适用场景 | 10–10⁴ 个用户的学生作品、小工具、试水付费墙、PoC | 任何"真的需要鉴权"的产品 |
| 部署 | `git push` 即可 | 需要服务器 CI/CD + 密钥管理 |

> **诚实的话**：这条 skill 不是 auth，是 paywall。任何登录状态都能被持有 dev tools 的人绕过——所以别拿它来保护真正的秘密或金钱。把它的"价值"想成"让普通人够不到付费内容"，别想成"挡住黑客"。

## 适用范围（先确认自己的场景落在哪档）

### 强烈建议使用

- 单文件 HTML + 静态托管（Render / CF Pages / Vercel / GitHub Pages / EdgeOne Pages）。
- 用户量小（< 几百），付费墙 + 内容解锁场景，预期被"绕过"的概率极低。
- 已经拒绝部署后端（或一个 Node Functions 都不愿上）。
- 项目里已经有"管理员"概念（一般是作者本人）——本 skill 不面向"用户自助注册"，所有账号由 admin 手工生成/发放。

### 视情况权衡使用

- 想加可选的"设备绑定" / "远程登出" 能力——本 skill 提供 Node Functions escape hatch（见第五节），仅在想要这些能力时才上。
- 想要"会员到期"语义—— schema 里有 `paidUntil` 字段，校验时只查 `<= now` 就过期；逻辑 8 行，但要写主 app 时自己加。
- 跨子域部署（如 `admin.app.com` 管 `app.com`）——需要严格按 `accountUrl=` / meta 同源约定，否则 CORS 卡你。

### 不建议使用 / 需要谨慎

- **任何需要"真鉴权"的场景**：银行、医疗、企业内部——按上面表格右侧走。
- **用户自助注册 + 邮件验证**：本 skill 完全没有发邮件/验证码能力。
- **GDPR / 隐私强合规场景**：`accounts.json` 是公开 JSON（hash 也是数据），部署到 CDN 等于把用户存在的事实公开化。
- **学校/企业内部"账号体系"**：绕过太容易，老师查不到谁登录了。
- **任何会被新闻曝光的"AI/数据/财务"项目**：付费墙挡不住，不要造错觉。

## 一、账号存储：sha256 + salt + JSON 文件

**现状**：admin HTML 直接在浏览器里 `crypto.subtle.digest('SHA-256', new TextEncoder().encode(salt + password))` 算哈希，把 `{u, h}` 数组写进 JSON；admin 自己就是一个静态页面，**不连任何后端**。

**做法**：设计 schema 时只放最小必要字段——用户名 `u`、哈希 `h`，外加一个 `version`（schema 升级用）和一个全局 `salt`（所有账号共用同一 salt）。**不要**在 JSON 里放过期时间、角色、备注——非必要字段放主 app 的 IndexedDB 缓存里更合适，账号文件改动越少越好。

**drop-in 参考（admin HTML 与主 app 共享）**：

```js
// 8 行解决：算哈希 + 校验
async function hashPassword(salt, password) {
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(salt + password));
  return Array.from(new Uint8Array(buf)).map(x => x.toString(16).padStart(2, '0')).join('');
}
async function verifyPassword(salt, password, expectedHash) {
  return (await hashPassword(salt, password)) === expectedHash.toLowerCase();
}
```

**schema（必须照搬）**：

```json
{
  "version": 1,
  "salt": "<?SALT?>",
  "accounts": [
    { "u": "alice", "h": "<64-char hex sha256>" },
    { "u": "bob",   "h": "<64-char hex sha256>" }
  ]
}
```

**验证**：

- 浏览器 console 跑 `hashPassword('<?SALT?>', 'alice')`，对比 admin HTML 显示的 `h`，hash 必须完全一致。
- 在 admin UI 新增一个账号 → 下载 `accounts.json` → 用任何 JSON 工具查看，`h` 字段必须是 64 字符小写 hex，无前缀 `0x`。

## 二、HTML 端登录校验：fetch + 重哈希

**现状**：主 app 登录时 `fetch(ACCOUNT_URL)` 拿到 JSON，浏览器里重算哈希比对。**没有 token**——登录态就是 `localStorage[<?APP_KEY?>_unlocked]` + 用户名。

**做法**：URL 解析按优先级 1→2→3（覆盖最常见的 3 种部署形态）：

```js
// 11 行：URL 解析三档 + file:// 降级
var ACCOUNT_URL = (function(){
  try {                                  // (1) URL 参数最优先（作者显式指定）
    var u = new URL(location.href).searchParams.get('accountUrl');
    if (u) return u.trim();
  } catch(_e) {}
  var meta = ((document.querySelector('meta[name="x-account-url"]')||{}).content||'').trim();
  if (meta) {                             // (2) meta 同源则用，跨源则降级
    try {
      var m = new URL(meta, location.href);
      if (m.origin === location.origin) return m.href;
    } catch(_e) {}
  }
  return location.origin + '/accounts.json';  // (3) 同源兜底
})();
```

**主 app HTML 头部加 meta**（允许本地 override）：

```html
<meta name="x-account-url" content="https://your.app.com/accounts.json">
```

**drop-in 登录函数**（主 app 顶部）：

```js
async function tryLogin(username, password) {
  try {
    const r = await fetch(ACCOUNT_URL, {cache: 'no-store'});
    const {salt, accounts} = await r.json();
    const account = accounts.find(a => a.u === username);
    if (!account) return {ok: false, err: '账号不存在'};
    const h = await hashPassword(salt, password);
    if (h !== account.h.toLowerCase()) return {ok: false, err: '密码错'};
    localStorage.setItem('<?APP_KEY?>_unlocked', JSON.stringify({u: username, ts: Date.now()}));
    return {ok: true, u: username};
  } catch (e) {
    return {ok: false, err: '无法连接账号服务（'+e.message+'）'};
  }
}
```

**验证**：用 Playwright 打开主 app，输入正确账号 → `localStorage[<?APP_KEY?>_unlocked]` 应有值；错误密码 → DOM 应出现「密码错」字样。

## 三、跨页/跨 tab 实时配置同步

**现状**：admin 改价格/微信号后，希望同浏览器开着的 N 个主 app tab 立即看到，不需要刷新。

**做法**：双保险 ——
1. **BroadcastChannel**（现代浏览器）：admin 写 `localStorage` 的同时 `postMessage({type:'cfg-update', ...})`。
2. **`storage` 事件**（兜底）：admin 写 localStorage 后，**其他 tab** 的 `window.addEventListener('storage', ...)` 会自动触发（同 tab 不触发）。

```js
// admin 端「写」的同步广播
function publishCfg(cfg) {
  localStorage.setItem('<?APP_KEY?>_cfg', JSON.stringify(cfg));
  try {
    if (typeof BroadcastChannel !== 'undefined') {
      new BroadcastChannel('<?APP_KEY?>_admin').postMessage({type:'cfg-update', ...cfg});
    }
  } catch(_e) {}
}

// 主 app 端「订阅」——注册两件事：BC + storage
function subscribeCfg(onChange) {
  // 首次加载读一次
  const saved = (() => { try { return JSON.parse(localStorage.getItem('<?APP_KEY?>_cfg')||'null'); } catch(_e){ return null; }})();
  if (saved) onChange(saved);
  // 双订阅
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

**验证**：开两个 tab，tab1 改价格 `19.9 → 29.9`，tab2 浏览器 console 监听收到 `cfg-update` 消息 < 500ms；3 秒内 tab2 UI 应显示新价格。

## 四、付费墙/会员门控（可选）

**现状**：登录成功后，主 app 要决定"哪些功能对登录用户开放"。两种最常见：
1. **软锁** — 未登录时显示一个引导卡片，按钮触发登录弹窗；登录后擦掉卡片露内容。
2. **硬门** — 未登录时整页替换为「请先登录」；登录后 `location.reload()`。

**做法**：跟登录态强耦合即可。一个状态机，3 个状态（`unknown` / `unlocked` / `locked`），主 app 启动时根据 `localStorage[<?APP_KEY?>_unlocked]` 推断：

```js
const PRO = { state: 'unknown', user: null };
(function bootstrapPro() {
  const saved = (() => { try { return JSON.parse(localStorage.getItem('<?APP_KEY?>_unlocked')||'null'); } catch(_e){ return null; }})();
  if (saved && saved.u && saved.ts) {
    PRO.state = 'unlocked'; PRO.user = saved.u;
  } else {
    PRO.state = 'locked';
  }
  // 让功能模块根据 PRO.state 决定渲染（硬门：`if (PRO.state!=='unlocked') showPaywall()`）
})();
```

会员到期（如有 `paidUntil`）— 在 `accounts.json` 给单个账号加 `"paidUntil": "2026-12-31"`，登录时同时校验：

```js
if (account.paidUntil && new Date(account.paidUntil) < new Date()) {
  return {ok: false, err: '会员已过期，请联系作者续费'};
}
```

**验证**：未登录打开主 app → 应看到付费墙；登录后 → 付费墙擦除；重新刷新页面 → 付费墙不会再出现（持久化生效）。

## 五、可选远端：Node Functions 登录/设备绑定

**现状**：如果你需要"每账号限 1 台设备" / "管理员远程踢人" / "最后登录时间审计"——纯前端做不到。**这是 escape hatch**：另起一个 Node Functions 端点 + KV 存储，admin HTML 加一个"设备管理" tab。

**什么时候该上**：

- ✅ 想跟客户说"你账号不能 5 个人共用"
- ✅ 想在客户投诉"我账号被盗"时能远程吊销
- ✅ 想知道"张三上周登录了几次"

**什么时候别上**：

- ❌ 上面三件事都不在乎——纯前端 hash 比 Node Functions 省 90% 维护成本
- ❌ 没部署过 EdgeOne Pages / Render 的 Web Service——先把基础部署搞定

**最小后端接口**（PoemGraph 已实现的，详见 `/Users/Zhuanz/Claude/PoemGraph/node-functions/`）：

| 端点 | 方法 | 鉴权 | 用途 |
|---|---|---|---|
| `/api/login` | POST | 无（公开，但需 username+password） | 主 app 调用，校验 sha256 + 检查 device 配额 |
| `/api/admin/list` | POST | `Authorization: Bearer ${config:adminToken}` | admin 拉所有账号的设备绑定状态 |
| `/api/admin/reset` | POST | 同上 | 清空某账号的设备列表 |
| `/api/admin/add-device` | POST | 同上 | 手工给某账号加设备 UUID |

KV 的 key 模式：`account:<username>` → `{u, h, devices: [uuid1, uuid2], lastSeen}`。

> 见下方「接入检查清单」第 7 步。

## 踩过的坑

> 这些是已经踩过的、写死在这里，别再踩一次。

1. **salt 多处重复导致 hash 不一致** — 把 `salt` 常量在 admin HTML 和主 app 各写一份 = 必踩。**约定**：salt 只在 `accounts.json` 出现（admin 写时填），admin HTML 和主 app **读** 它的 `salt` 字段，不硬编码任何 salt。万一两份代码已经冲突，先 grep `salt` 看有多少份，再统一为"读 JSON 的 salt"。

2. **`localStorage.unlocked=true` 不是真登录** — 任何持有浏览器 dev tools 的人都改得了。**承认**这件事：admin UI 顶部明确写「**零信任客户端，仅作软门**」；主 app README 写「**这不是 auth，是 paywall**」。别自己骗自己。

3. **跨域 `accounts.json` CORS 坑** — Render/Cloudflare Pages 默认同源 OK，但 GitHub Pages + 自定义域 + 跨子域部署时 fetch 被拦。**对策**：要么保证 admin 和主 app 同 origin，要么给 `accounts.json` 加 `_headers` 配置 `Access-Control-Allow-Origin: *`（注意：暴露用户存在性，权衡）。最容易的方案：admin 和主 app 部署在同一站点的同一域名下，**0 CORS 0 麻烦**。

4. **`file://` 半残** — WebCrypto 在 `file://` 协议下多数浏览器拒绝（`crypto.subtle === undefined`）。admin HTML 必须 detect 协议：
   ```js
   if (location.protocol === 'file:') {
     document.body.insertAdjacentHTML('afterbegin',
       '<div class="warn">请用 http://localhost 打开，不要直接双击 file://</div>');
   }
   ```
   **永远**给用户"用 python -m http.server 起本地服务"的明确提示，而不是让他们摸索。

5. **密码明文落 localStorage** — 调试时方便 → 上线忘删 = 重大事故。**禁止**：localStorage 只放 `{u, ts}` 和 `paidUntil`，表单 submit 后立即 `form.reset()`。CI 加一条 lint：`grep -rE 'localStorage.setItem.*\\bpwd|password' src/` 必须为空。

6. **IndexedDB 已有数据 vs 新账号 id 冲突**（WhiteBoard 类项目特别注意）— `unlocked=true` 不应该让 IndexedDB 记录 ownership 变；写每步操作要再 check 一次角色，**不依赖**登录态缓存。比如 IndexedDB 里有个 `whiteboard.doc`，登录态变化时别无脑把它转"我的"，而是按 `?doc=` URL 参数决定所有权；登录态只决定"能不能写"。

7. **「用户要真鉴权，我给的是玩具鉴权」期望错配** — 立项时一定在 README 写「这不是 auth，是 paywall」，免得哪天用户拿 token 改权限产生纠纷。**唯一能号称"真鉴权"的时刻**：当且仅当你接了第五节的 Node Functions 端点 + JWT/签名 token + 服务器侧吊销。否则就别用"auth"这个词。

8. **`accounts.json` 写了 placeholder 账号被 commit** — `accounts.json.example` 进 git，真实 `accounts.json` 必须 `.gitignore`。否则你 stub 写的 demo 密码的 hash 直接成为线上弱口令。**约定**：仓库有两个文件，example 公开、真实 gitignore，admin UI 默认 fetch 真实文件的 URL（同站部署），example 仅给首次 git clone 的开发者当 README 参考。

## 接入检查清单

> 这 8 步对 PoemGraph / WhiteBoard / 其他纯 HTML 项目都一样，按顺序走完。

1. **复制 admin** — 把 `templates/account-admin.html` 拷到项目根，把 12 个 `<?...?>` 占位符全部替换为你的项目实际值（最关键的是 `<?APP_KEY?>`，决定 LS key 和 BroadcastChannel 名）。
2. **准备 `accounts.json.example`** — 提交到仓库根；UI 默认 `fetch ${origin}/accounts.json`；**真实 `accounts.json` 必须 `.gitignore`**，避免 placeholder 账号被 commit。
3. **主 app HTML 头部加 3 个 meta**：
   ```html
   <meta name="x-account-url" content="./accounts.json">
   <meta name="x-<?APP_KEY?>-price" content="<?DEFAULT_PRICE?>">
   <meta name="x-<?APP_KEY?>-wx" content="<?OWNER_WECHAT?>">
   ```
4. **主 app 顶部加 login 函数 + PRO bootstrap** — 复制第二节的 drop-in，绑定到登录表单的 submit。
5. **接 BroadcastChannel + storage 订阅** — 复制第三节的 drop-in，开 2 个 tab 验证 admin 改价格 → 主 app 实时更新（< 500ms）。
6. **(可选)付费墙** — 第四节 3 状态机；按你页面需要决定是软锁（卡片替换）还是硬门（整页替换）。
7. **(可选)Node Functions 端点** — 仅当你要远程踢人/设备绑定/审计时才做；PoemGraph 已实现 `/api/login + /api/admin/{list,reset,add-device}`，复制 `node-functions/` 目录到你的项目根，按 EdgeOne Pages / Render 文档部署。
8. **部署验证** — Render / CF Pages 的 `_headers` 必须为 `accounts.json` 加 `Content-Type: application/json`；如果 admin HTML 在不同子域访问主站，加 `Access-Control-Allow-Origin: *`（警告：暴露用户数据，权衡）。

## 变更记录

| 日期 | 变更 |
|---|---|
| 2026-07-13 | v1.0.0：从 PoemGraph `account-admin.html` 抽出通用 skill。4 文件（SKILL.md / INTEGRATION.md / templates/account-admin.html / templates/accounts.json.example）。覆盖存储 / 校验 / 跨 tab 同步 / 付费墙 / 可选远端 五节 + 8 条踩坑 + 8 步接入清单。 |
