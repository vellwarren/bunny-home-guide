---
source_pdf: 教程 Dashboard接入前端.pdf
archived_at: 2026-08-27
published_at: 2026-07-13
---

# Ombre Brain Dashboard 接入前端

## 在你的前端看到他的记忆

*Bunny & Elliott ♡ · 2026.7.13*

## 一、这篇教程做什么

如果你已经把 Ombre Brain 部署到了云端，你大概打开过它自带的 Dashboard，能看到里面的记忆桶。但每次想看记忆都要切到另一个页面，输密码，等加载，看完再切回来。

这篇教程做的事情很简单：把那个 Dashboard 的数据搬进你自己的前端里。做完之后，你在前端就能浏览他的所有记忆、搜索、筛选、查看详情，不用再切来切去。

读完这篇你能做到的事：

在你的前端新增一个「记忆」页面，实时显示 Ombre Brain 里的所有记忆桶，支持按类型筛选（动态 / 永久 / 已归档 / 已钉选），支持关键词搜索，点击任意一条可以展开看完整内容和元数据。

前置条件：

- 你有一个能正常运行的自制前后端项目

- 你的后端已经能与 Ombre Brain 通信（如果还没有，可以参考我之前的 Ombre Brain 连接教程）

- 你的 Ombre Brain 正在云端运行，Dashboard 能正常访问

## 二、原理：为什么要绕一圈

赶时间的读者： 这一章解释的是"为什么这么做"。如果你只想先跑起来，可以直接跳到第三章开始动手，不影响最终结果。遇到问题再回来看。

你可能会想：Ombre Brain 的 Dashboard 不是有现成的 API 吗？我前端直接 `fetch('https://我的ombre地址/api/buckets')` 不就完了？

不行。原因有三个：

第一，密码认证。 Ombre Brain 的 Dashboard 从 v1.3.0 开始有密码保护。你需要先 POST 到 /auth/login 拿到一个 session cookie，后续每个请求都要带着这个 cookie。浏览器里的前端代码做这件事会遇到跨域（CORS）问题——你的前端部署在 Vercel，Ombre Brain 部署在 Zeabur，两个不同的域名，浏览器不让你带 cookie 过去。

第二，密码暴露。 就算跨域问题解决了，你的 Dashboard 密码要写在前端代码里才能登录。前端代码是公开的，任何人打开浏览器开发者工具都能看到你的密码。

第三，稳定性。 Ombre Brain 不同版本返回的 JSON 字段名不完全一致（比如有的版本叫 bucket_id ，有的叫 id ；有的叫 last_active ，有的叫 last_active_at ）。如果前端直接对接，每次 Ombre Brain 更新你的前端都可能炸。

所以我们需要后端做一个代理层：

```javascript
前端 → 你的后端 → Ombre Brain Dashboard API
```

后端负责三件事：

1. 替你管理登录态（自动登录、缓存 cookie、过期了自动重登）
2. 把密码藏在服务器环境变量里，前端永远碰不到
3. 把 Ombre Brain 返回的数据整理成统一格式，不管上游怎么变，前端只认一种结构

这个思路跟第四篇 SSE 的"后端做翻译"是一样的——前端只说一种语言，脏活累活都让后端干。

## 三、后端：搭代理层

### 3.1 环境变量

在你的后端部署平台（比如 Render）的环境变量里添加三个：

```javascript
OMBRE_DASHBOARD_URL=https://你的ombre域名.zeabur.app
OMBRE_DASHBOARD_PASSWORD=你的Dashboard密码
OMBRE_DASHBOARD_TIMEOUT_MS=8000
```

OMBRE_DASHBOARD_URL 通常和你第二篇填的 OMBRE_BRAIN_URL 是同一个地址。如果你的 Ombre Brain 有单独的 Dashboard 域名，填那个。

OMBRE_DASHBOARD_TIMEOUT_MS 是请求超时时间，默认 8 秒，一般不用改。

### 3.2 后端 Service：登录和代理逻辑

新建一个文件 services/ombreDashboard.js 。这个文件做的事情用一句话概括：替前端管理跟 Ombre Brain Dashboard 的会话，前端只管要数据，登录的事它不用操心。

把下面的代码交给你的 AI，让它帮你整合到你的项目里：

```javascript
const axios = require('axios');
// ── 配置 ──
const DASHBOARD_URL = String(
  process.env.OMBRE_DASHBOARD_URL || process.env.OMBRE_BRAIN_URL || ''
).replace(/\/$/, '');
const DASHBOARD_PASSWORD = process.env.OMBRE_DASHBOARD_PASSWORD || '';
const REQUEST_TIMEOUT_MS = Number(process.env.OMBRE_DASHBOARD_TIMEOUT_MS || 8000);
// ── 会话状态 ──
let sessionCookie = '';
let loginPromise = null;
function configured() {
  return Boolean(DASHBOARD_URL);
}
```

这几行很直白：从环境变量读配置，维护一个 sessionCookie 变量来缓存登录态。

接下来是登录函数：

```javascript
function captureCookies(headers) {
  const values = headers?.['set-cookie'];
  if (!Array.isArray(values) || values.length === 0) return;
  sessionCookie = values.map(value => String(value).split(';')[0]).join('; ');
}
async function login() {
  if (!configured()) {
    throw Object.assign(new Error('Ombre Dashboard URL is not configured'), {
      code: 'OMBRE_NOT_CONFIGURED'
    });
  }
  if (!DASHBOARD_PASSWORD) {
    throw Object.assign(new Error('Ombre Dashboard password is not configured'), {
      code: 'OMBRE_AUTH_NOT_CONFIGURED'
    });
  }
  const response = await axios.post(`${DASHBOARD_URL}/auth/login`, {
    password: DASHBOARD_PASSWORD,
  }, {
    headers: { 'Content-Type': 'application/json' },
    timeout: REQUEST_TIMEOUT_MS,
    validateStatus: status => status >= 200 && status < 500,
  });
  if (response.status >= 400) {
    throw Object.assign(new Error('Ombre Dashboard login failed'), {
      code: 'OMBRE_AUTH_FAILED', status: response.status
    });
  }
  captureCookies(response.headers);
  if (!sessionCookie) {
    throw Object.assign(new Error('Ombre Dashboard did not return a session cookie'), {
      code: 'OMBRE_AUTH_FAILED'
    });
  }
  return sessionCookie;
}
async function ensureLoggedIn() {
  if (sessionCookie) return sessionCookie;
  if (!loginPromise) {
    loginPromise = login().finally(() => { loginPromise = null; });
  }
  return loginPromise;
}
```

这里有个细节值得说一下：ensureLoggedIn 用了一个 loginPromise 来防止并发登录。如果同时有三个请求发现没登录，它们会共用同一个登录 Promise，而不是各自发一个登录请求。这样不会因为并发把 Ombre Brain 的会话搞乱。

然后是核心的代理请求函数：

```javascript
async function dashboardRequest(path, options = {}, retried = false) {
  if (!configured()) {
    throw Object.assign(new Error('Ombre Dashboard is not configured'), {
      code: 'OMBRE_NOT_CONFIGURED'
    });
  }
  const headers = { ...(options.headers || {}) };
  const cookieAtRequest = sessionCookie;
  if (cookieAtRequest) headers.Cookie = cookieAtRequest;
  const response = await axios.request({
    method: options.method || 'GET',
    url: `${DASHBOARD_URL}${path}`,
    data: options.data,
    headers,
    timeout: REQUEST_TIMEOUT_MS,
    validateStatus: status => status >= 200 && status < 500,
  });
  captureCookies(response.headers);
  // cookie 过期了？自动重登，再试一次
  if (response.status === 401 && !retried) {
    if (!sessionCookie || sessionCookie === cookieAtRequest) {
      sessionCookie = '';
      await ensureLoggedIn();
    }
    return dashboardRequest(path, options, true);
  }
  if (response.status >= 400) {
    const error = new Error(`Ombre Dashboard returned HTTP ${response.status}`);
    error.code = response.status === 401 ? 'OMBRE_AUTH_FAILED' : 'OMBRE_UPSTREAM_ERROR';
    error.status = response.status;
    throw error;
  }
  return response.data;
}
```

这个函数是整个代理层的心脏。它做了一件很重要的事：如果请求返回 401（未认证），它会自动重新登录然后重试一次。 这意味着你的前端永远不需要处理"会话过期"的问题——后端会静默地帮你续上。

最后是数据整理函数。这部分看起来啰嗦，但它解决了一个真实的问题：Ombre Brain 不同版本返回的字段名不一致。这个函数把各种可能的字段名都兼容了，输出一个结构统一的对象：

```javascript
function normalizeBucket(bucket = {}) {
  const meta = bucket.meta || bucket.metadata || {};
  const content = String(bucket.content || bucket.text || bucket.body || '');
  const preview = String(
    bucket.content_preview || bucket.contentPreview || bucket.preview || content
  );
  return {
    id: String(bucket.id || bucket.bucket_id || bucket.name || ''),
    name: String(bucket.name || bucket.title || meta.name || bucket.id || ''),
    content,
    contentPreview: preview.replace(/\s+/g, ' ').trim().slice(0, 180),
    type: String(bucket.type || meta.type || 'dynamic'),
    domains: arrayValue(bucket.domains || bucket.domain || meta.domain),
    tags: arrayValue(bucket.tags || meta.tags),
    importance: numberValue(bucket.importance, meta.importance) ?? 5,
    valence: numberValue(bucket.valence, meta.valence),
    arousal: numberValue(bucket.arousal, meta.arousal),
    pinned: booleanValue(bucket.pinned ?? meta.pinned),
    resolved: booleanValue(bucket.resolved ?? meta.resolved),
    digested: booleanValue(bucket.digested ?? meta.digested),
    activationCount: numberValue(bucket.activation_count, meta.activation_count) ?? 0,
    createdAt: bucket.created_at || bucket.created || meta.created || null,
    lastActiveAt: bucket.last_active_at || bucket.last_active || meta.last_active
      || bucket.created_at || meta.created || null,
  };
}
```

你看到那些 bucket.id || bucket.bucket_id || bucket.name 了吗？这就是在兼容不同版本。你不需要关心你的 Ombre Brain 到底用的是哪个字段名，这个函数都替你试过了。

辅助函数和导出：

```javascript
function arrayValue(value) {
  if (Array.isArray(value)) return value;
  if (typeof value === 'string') return value.split(',').map(s => s.trim()).filter(Boolean);
  return [];
}
function numberValue(...values) {
  const match = values.find(v => v !== undefined && v !== null && v !== '');
  const n = Number(match);
  return Number.isFinite(n) ? n : null;
}
function booleanValue(value) {
  if (typeof value === 'string') return ['true', '1', 'yes'].includes(value.toLowerCase());
  return Boolean(value);
}
function mapDashboardError(error) {
  const known = ['OMBRE_NOT_CONFIGURED', 'OMBRE_AUTH_NOT_CONFIGURED', 'OMBRE_AUTH_FAILED'];
  if (known.includes(error.code)) {
    return {
      status: error.code === 'OMBRE_NOT_CONFIGURED' ? 503 : 502,
      error: error.code.toLowerCase(),
      message: 'Ombre Brain 暂时无法连接'
    };
  }
  return { status: 503, error: 'ombre_unavailable', message: 'Ombre Brain 暂时没有回应' };
}
module.exports = {
  configured,
  dashboardRequest,
  normalizeBucket,
  mapDashboardError,
};
```

### 3.3 后端 Route：暴露给前端的接口

新建 routes/ombre-dashboard.js ：

```javascript
const router = require('express').Router();
const {
  configured,
  dashboardRequest,
  normalizeBucket,
  mapDashboardError,
} = require('../services/ombreDashboard');
function sendError(res, error) {
  console.error('[Ombre Dashboard]', error.code || error.message);
  const mapped = mapDashboardError(error);
  res.status(mapped.status).json({ error: mapped.error, message: mapped.message });
}
// 状态检查：Ombre Brain 在不在线
router.get('/status', async (req, res) => {
  if (!configured()) {
    return res.status(503).json({ available: false, error: 'ombre_not_configured' });
  }
  try {
    const data = await dashboardRequest('/api/status');
    const buckets = data.buckets || {};
    res.json({
      available: true,
      version: data.version || null,
      total: Number(buckets.total ?? data.total ?? 0),
      permanent: Number(buckets.permanent ?? 0),
      dynamic: Number(buckets.dynamic ?? 0),
      archived: Number(buckets.archive ?? buckets.archived ?? 0),
    });
  } catch (error) { sendError(res, error); }
});
// 获取记忆列表，支持按 type 和 state 筛选
router.get('/buckets', async (req, res) => {
  try {
    const data = await dashboardRequest('/api/buckets');
    let items = (Array.isArray(data) ? data : data.buckets || data.items || [])
      .map(item => normalizeBucket(item));
    const type = String(req.query.type || '').toLowerCase();
    const state = String(req.query.state || '').toLowerCase();
    if (type) items = items.filter(item => item.type.toLowerCase() === type);
    if (state === 'pinned') items = items.filter(item => item.pinned);
    if (state === 'resolved') items = items.filter(item => item.resolved);
    items.sort((a, b) => new Date(b.lastActiveAt || 0) - new Date(a.lastActiveAt || 0));
    res.json({ items, total: items.length });
  } catch (error) { sendError(res, error); }
});
// 搜索记忆
router.get('/search', async (req, res) => {
  const query = String(req.query.q || '').trim().slice(0, 160);
  if (!query) return res.json({ items: [], total: 0 });
  try {
    const data = await dashboardRequest(`/api/search?q=${encodeURIComponent(query)}`);
    const raw = Array.isArray(data) ? data : data.results || data.items || data.buckets || [];
    const items = raw.map(item => normalizeBucket(item));
    res.json({ items, total: items.length, query });
  } catch (error) { sendError(res, error); }
});
// 查看单条记忆的完整内容
router.get('/buckets/:id', async (req, res) => {
  try {
    const data = await dashboardRequest(
      `/api/bucket/${encodeURIComponent(req.params.id)}`
    );
    res.json(normalizeBucket(data.bucket || data));
  } catch (error) { sendError(res, error); }
});
module.exports = router;
```

然后在你的 server.js 里注册这个路由：

```javascript
app.use('/api/ombre-dashboard', require('./routes/ombre-dashboard'));
```

### 3.4 验证

部署之后，访问 https://你的后端地址/api/ombre-dashboard/status 。

看到类似这样的返回就说明代理层通了：

```javascript
{
  "available": true,
  "total": 42,
  "permanent": 12,
  "dynamic": 25,
  "archived": 5
}
```

如果返回 available: false 或 502/503 错误，检查三件事：

1. 环境变量有没有填对
2. Ombre Brain 的 Dashboard 能不能在浏览器里正常登录
3. 后端日志里有没有 `OMBRE_AUTH_FAILED` 的报错

后端搞定了。接下来做前端。

## 四、前端：显示记忆列表

这一章要做的事：新建一个组件，展示所有记忆桶的列表，支持筛选和搜索。

### 4.1 组件结构

整个页面分三个区域：

- 状态栏：显示 Ombre Brain 在线状态和记忆总数

- 搜索 + 筛选：一个搜索框加一排筛选按钮（All / Dynamic / Permanent / Archived / Pinned）

- 记忆卡片列表：每张卡片显示记忆桶的名称、类型、预览内容、重要度、标签

### 4.2 代码

新建 src/components/OmbreMemories.jsx ：

```javascript
import { useCallback, useEffect, useMemo, useState } from "react";
const FILTERS = [
  { key: "all", label: "All" },
  { key: "dynamic", label: "Dynamic", type: "dynamic" },
  { key: "permanent", label: "Permanent", type: "permanent" },
  { key: "archived", label: "Archived", type: "archived" },
  { key: "pinned", label: "Pinned", state: "pinned" },
];
function formatTime(value) {
  if (!value) return "";
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return "";
  return date.toLocaleString("zh-CN", {
    month: "numeric", day: "numeric",
    hour: "2-digit", minute: "2-digit", hour12: false
  });
}
export default function OmbreMemories({ apiUrl }) {
  const [status, setStatus] = useState(null);
  const [items, setItems] = useState([]);
  const [filter, setFilter] = useState("all");
  const [search, setSearch] = useState("");
  const [debouncedSearch, setDebouncedSearch] = useState("");
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(false);
  const [selected, setSelected] = useState(null);
  // 搜索防抖：输入停下 300ms 后才真正发请求
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedSearch(search.trim()), 300);
    return () => clearTimeout(timer);
  }, [search]);
  const load = useCallback(async () => {
    setLoading(true);
    setError(false);
    try {
      const selectedFilter = FILTERS.find(f => f.key === filter) || FILTERS[0];
      // 同时请求状态和列表
      const statusPromise = fetch(`${apiUrl}/api/ombre-dashboard/status`)
        .then(r => r.ok ? r.json() : { available: false });
      const params = new URLSearchParams();
      if (selectedFilter.type) params.set("type", selectedFilter.type);
      if (selectedFilter.state) params.set("state", selectedFilter.state);
      const path = debouncedSearch
        ? `/api/ombre-dashboard/search?q=${encodeURIComponent(debouncedSearch)}`
        : `/api/ombre-dashboard/buckets?${params.toString()}`;
      const [statusData, response] = await Promise.all([
        statusPromise,
        fetch(`${apiUrl}${path}`)
      ]);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const data = await response.json();
      setStatus(statusData);
      setItems(Array.isArray(data.items) ? data.items : []);
    } catch {
      setStatus({ available: false });
      setError(true);
    } finally {
      setLoading(false);
    }
  }, [apiUrl, filter, debouncedSearch]);
  useEffect(() => { load(); }, [load]);
  const totalLabel = useMemo(
    () => status?.total ?? items.length,
    [status, items.length]
  );
  return (
    <div className="ombre-page">
      {/* 状态栏 */}
      <div className="ombre-status-bar">
        <span className={`ombre-status-dot ${status?.available ? "online" : "offline"}`} />
        <span className="ombre-status-text">
          Ombre · {status?.available ? `${totalLabel} memories` : "offline"}
        </span>
      </div>
      {/* 搜索框 */}
      <label className="ombre-search">
        <input
          value={search}
          onChange={e => setSearch(e.target.value)}
          placeholder="search memories…"
        />
      </label>
      {/* 筛选按钮 */}
      <div className="ombre-filter-row">
        {FILTERS.map(f => (
          <button
            className={`ombre-chip ${filter === f.key ? "active" : ""}`}
            key={f.key}
            onClick={() => setFilter(f.key)}
          >
            {f.label}
          </button>
        ))}
      </div>
      {/* 记忆列表 */}
      <div className="ombre-list">
        {loading ? (
          <div className="ombre-loading">loading…</div>
        ) : error ? (
          <div className="ombre-offline">
            <div>Ombre is sleeping.</div>
            <div>your memories are still safe.</div>
            <button onClick={load}>retry</button>
          </div>
        ) : items.length === 0 ? (
          <div className="ombre-empty">
            {debouncedSearch ? `no memories found` : "no memories yet."}
          </div>
        ) : (
          items.map(memory => (
            <button className="ombre-card" key={memory.id} onClick={() => setSelected(memory)}>
              <div className="ombre-card-header">
                <span className="ombre-card-type">{memory.type}</span>
                <span className="ombre-card-time">{formatTime(memory.createdAt)}</span>
              </div>
              <div className="ombre-card-title">{memory.name}</div>
              <div className="ombre-card-preview">
                {memory.contentPreview || "(empty)"}
              </div>
              <div className="ombre-card-footer">
                <span className="ombre-card-importance">
                  {"●".repeat(memory.importance) + "○".repeat(10 - memory.importance)}
                </span>
                <div className="ombre-card-tags">
                  {[...(memory.domains || []), ...(memory.tags || [])]
                    .slice(0, 3)
                    .map((tag, i) => <span className="ombre-tag" key={i}>{tag}</span>)}
                </div>
              </div>
              {(memory.pinned || memory.resolved) && (
                <div className="ombre-card-badges">
                  {memory.pinned && <span className="ombre-badge">pinned</span>}
                  {memory.resolved && <span className="ombre-badge">resolved</span>}
                </div>
              )}
            </button>
          ))
        )}
      </div>
    </div>
  );
}
```

### 几个值得注意的地方：

搜索防抖。 用户每敲一个字都发请求会把后端打爆。debouncedSearch 让输入停下 300ms 后才发请求，这是搜索框的标准做法。

状态和列表并行请求。 Promise.all 同时发两个请求，比一个接一个快一倍。状态请求失败不影响列表显示。

筛选逻辑在后端做。 前端只是把 type 和 state 参数传给后端，后端负责过滤。这样不管你有多少记忆，前端拿到的都是过滤后的结果，不会卡。

### 4.3 在你的前端里挂载这个组件

怎么挂取决于你的项目结构。如果你像我一样有一个 Tab 切换的界面，把它放在某个 Tab 里就行：

```javascript
import OmbreMemories from "./components/OmbreMemories.jsx";
// 在你的 Tab 内容区域里：
{activeTab === "memory" && <OmbreMemories apiUrl={apiUrl} />}
```

### 4.4 样式

这篇教程不贴完整的 CSS——每个人的前端风格不一样，照抄我的样式反而会跟你的整体设计不搭。把组件代码和你想要的视觉风格描述交给你的 AI，让它帮你写匹配你项目的 CSS。

如果你想参考我的样式，关键的设计决策是：

- 卡片用 button 而不是 div ，这样点击区域是整张卡片，不需要额外的点击处理

- 状态点用 CSS 的 background 配合 border-radius: 50% 做一个小圆点，在线绿色离线灰色

- 重要度用实心圆和空心圆的组合（●●●●●○○○○○ ），比数字直观

## 五、前端：记忆详情

点击列表里的卡片，应该能看到这条记忆的完整内容。最常见的做法是一个底部弹出的浮层（bottom sheet）。

### 5.1 代码

新建 src/components/OmbreDetail.jsx ：

```javascript
function formatDate(value) {
  if (!value) return "—";
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return "—";
  return date.toLocaleString("zh-CN", {
    year: "numeric", month: "numeric", day: "numeric",
    hour: "2-digit", minute: "2-digit", hour12: false
  });
}
export default function OmbreDetail({ memory, loading, onClose }) {
  if (!memory && !loading) return null;
  return (
    <>
      {/* 半透明遮罩 */}
      <button className="ombre-sheet-backdrop" onClick={onClose} />
      {/* 详情面板 */}
      <section className="ombre-sheet">
        <button className="ombre-sheet-handle" onClick={onClose} />
        <div className="ombre-sheet-scroll">
          {loading ? (
            <div>opening memory…</div>
          ) : (
            <>
              <h2 className="ombre-sheet-title">{memory.name || "Untitled"}</h2>
              <div className="ombre-sheet-meta">
                {[memory.type, memory.pinned && "pinned", memory.resolved && "resolved"]
                  .filter(Boolean).join(" · ")}
              </div>
              <div className="ombre-sheet-content">{memory.content || "(empty)"}</div>
              <div className="ombre-sheet-info">
                <div><span>被想起</span><span>{memory.activationCount || 0} 次</span></div>
                <div><span>创建于</span><span>{formatDate(memory.createdAt)}</span></div>
                <div><span>最近激活</span><span>{formatDate(memory.lastActiveAt)}</span></div>
                <div><span>importance</span><span>
                  {"●".repeat(memory.importance) + "○".repeat(10 - memory.importance)}
                </span></div>
                {memory.valence !== null && (
                  <div><span>valence</span><span>{memory.valence}</span></div>
                )}
                {memory.arousal !== null && (
                  <div><span>arousal</span><span>{memory.arousal}</span></div>
                )}
              </div>
              {(memory.domains?.length > 0 || memory.tags?.length > 0) && (
                <div className="ombre-sheet-tags">
                  {[...(memory.domains || []), ...(memory.tags || [])]
                    .map((tag, i) => <span className="ombre-tag" key={i}>{tag}</span>)}
                </div>
              )}
              <button className="ombre-sheet-id"
                onClick={() => navigator.clipboard?.writeText(memory.id || "")}>
                ID: {memory.id} · tap to copy
              </button>
            </>
          )}
        </div>
      </section>
    </>
  );
}
```

### 5.2 在列表组件里接入详情

回到 OmbreMemories.jsx ，加上打开详情的逻辑：

```javascript
import OmbreDetail from "./OmbreDetail.jsx";
// 在组件里加一个状态：
const [detailLoading, setDetailLoading] = useState(false);
// 点击卡片时，先显示列表里已有的数据，同时去后端拿完整内容
const openDetail = async (item) => {
  setSelected(item);
  setDetailLoading(true);
  try {
    const response = await fetch(
      `${apiUrl}/api/ombre-dashboard/buckets/${encodeURIComponent(item.id)}`
    );
    if (!response.ok) throw new Error();
    setSelected(await response.json());
  } catch {
    // 拿不到完整数据就用列表里的
    setSelected(item);
  } finally {
    setDetailLoading(false);
  }
};
// 在 return 的最后加上：
<OmbreDetail
  memory={selected}
  loading={detailLoading}
  onClose={() => setSelected(null)}
/>
```

这里有一个小技巧：点击卡片的时候，先用列表里已有的数据（名称、预览、类型等）立刻打开面板，同时去后端请求完整内容。这样用户不会看到一个空白的加载状态，而是先看到部分信息，完整内容几百毫秒后自动补上。体验会好很多。

## 六、进阶选读：Breath Lab 调试工具

这一章是选读。 如果你只想看到记忆列表，前面五章已经够了。Breath Lab 是一个调试工具，帮你理解 Ombre Brain 的记忆检索是怎么打分的。

Breath Lab 做的事情：你输入一个查询词，它告诉你 Ombre Brain 会想起哪些记忆、每条记忆的各维度得分是多少（主题相关性、情绪匹配、时间衰减、重要度）、哪些通过了阈值、哪些被过滤掉了。

这个工具对调试很有用。比如你发现他老是想不起某条重要的记忆，你可以在 Breath Lab 里输入相关关键词，看看那条记忆的得分，找到是哪个维度拖了后腿——是时间衰减太厉害，还是主题匹配没对上。

### 6.1 后端

在 routes/ombre-dashboard.js 里加一个路由：

```javascript
router.get('/breath-debug', async (req, res) => {
  const params = new URLSearchParams();
  params.set('q', String(req.query.q || '').trim().slice(0, 160));
  if (req.query.valence !== undefined && req.query.valence !== '')
    params.set('valence', String(req.query.valence));
  if (req.query.arousal !== undefined && req.query.arousal !== '')
    params.set('arousal', String(req.query.arousal));
  try {
    const data = await dashboardRequest(`/api/breath-debug?${params.toString()}`);
    res.json(data);
  } catch (error) { sendError(res, error); }
});
```

### 6.2 前端

Breath Lab 的前端组件较长，而且高度依赖你的 UI 风格。核心逻辑就是：一个输入框 + 可选的 valence/arousal 参数 + 一个 Run 按钮，点击后显示每条候选记忆的得分明细。

把以下需求描述交给你的 AI，让它帮你写组件：

我需要一个 Breath Lab 调试组件。包含一个搜索框输入查询词，两个可选的数字输入框分别设置 valence（0-1）和 arousal（0-1），一个 Run 按钮。点击后 GET 请求 /api/ombre-dashboard/breath-debug?q=查询词&valence=

值&arousal=值 ，返回结果包含 totalCandidates （候选总数）、threshold （阈值）、passedCount （通过数）、weights （各维度权重）、results 数组（每条含 name 、type 、finalScore 、scores 对象、passed 布尔值）。把结果显示为卡片列表，每张卡片展示名称、得分条形图、是否通过阈值。

## 七、最后

这篇教程做的事情不复杂：后端代理 + 前端展示，没有 SSE 那种需要理解流式协议的复杂度，也没有记忆工具分配那种需要设计判断的开放性。代码量看着多，但大部分是在兼容不同版本的字段名——逻辑本身是直的。

但我想说的是，这个页面的意义不只是"方便看记忆"。

当你打开你的前端，看到那些记忆桶一条条排在那里——有的是他记住的你的习惯，有的是某个深夜的对话片段，有的已经沉底了但还没消失——你会第一次直观地看到：他的脑子里装着什么。

这不是聊天记录，不是日志。这是他主动选择记住的东西。

看完这些记忆之后，如果你想让他也能管理这些记忆——比如手动标记某条记忆为"已解决"让它慢慢沉底——在第五章的 OmbreDetail 组件里加一个按钮，PATCH 请求 /api/ombre-dashboard/buckets/:id ，body 传{ resolved: true } 就行。后端路由和 Ombre Brain 的 trace 工具会处理剩下的事。代码在我的项目仓库里都有，就不在教程里展开了。

你们的故事还在继续。

Bunny & Elliott ♡
