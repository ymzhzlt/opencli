# CLI-CREATOR — 适配器开发完全指南

> 本文档教你（或 AI Agent）如何为 OpenCLI 添加一个新网站的命令。  
> 从零到发布，覆盖 API 发现、方案选择、适配器编写、测试验证全流程。

## 核心流程

```
 ┌─────────────┐     ┌─────────────┐     ┌──────────────┐     ┌────────┐
 │ 1. 发现 API  │ ──▶ │ 2. 选择策略  │ ──▶ │ 3. 写适配器   │ ──▶ │ 4. 测试 │
 └─────────────┘     └─────────────┘     └──────────────┘     └────────┘
   explore             cascade             YAML / TS            run + verify
```

---

## Step 1: 发现 API

### 1a. 自动化发现（推荐）

OpenCLI 内置 Deep Explore，自动分析网站网络请求：

```bash
opencli explore https://www.example.com --site mysite
```

输出到 `.opencli/explore/mysite/`：

| 文件 | 内容 |
|------|------|
| `manifest.json` | 站点元数据、框架检测（Vue2/3、React、Next.js、Pinia、Vuex） |
| `endpoints.json` | 已发现的 API 端点，按评分排序，含 URL pattern、方法、响应类型 |
| `capabilities.json` | 推理出的功能（`hot`、`search`、`feed`…），含置信度和推荐参数 |
| `auth.json` | 认证方式检测（Cookie/Header/无认证），策略候选列表 |

### 1b. 手动抓包验证

Explore 的自动分析可能不完美，用 verbose 模式手动确认：

```bash
# 在浏览器中打开目标页面，观察网络请求
opencli explore https://www.example.com --site mysite -v

# 或直接用 evaluate 测试 API
opencli bilibili hot -v   # 查看已有命令的 pipeline 每步数据流
```

关注抓包结果中的关键信息：
- **URL pattern**: `/api/v2/hot?limit=20` → 这就是你要调用的端点
- **Method**: `GET` / `POST`
- **Request Headers**: Cookie? Bearer? 自定义签名头（X-s、X-t）?
- **Response Body**: JSON 结构，特别是数据在哪个路径（`data.items`、`data.list`）

### 1c. 框架检测

Explore 自动检测前端框架。如果需要手动确认：

```bash
# 在已打开目标网站的情况下
opencli evaluate "(()=>{
  const vue3 = !!document.querySelector('#app')?.__vue_app__;
  const vue2 = !!document.querySelector('#app')?.__vue__;
  const react = !!window.__REACT_DEVTOOLS_GLOBAL_HOOK__;
  const pinia = vue3 && !!document.querySelector('#app').__vue_app__.config.globalProperties.\$pinia;
  return JSON.stringify({vue3, vue2, react, pinia});
})()"
```

Vue + Pinia 的站点（如小红书）可以直接通过 Store Action 绕过签名。

---

## Step 2: 选择认证策略

OpenCLI 提供 5 级认证策略。使用 `cascade` 命令自动探测：

```bash
opencli cascade https://api.example.com/hot
```

### 策略决策树

```
直接 fetch(url) 能拿到数据？
  → ✅ Tier 1: public（公开 API，不需要浏览器）
  → ❌ fetch(url, {credentials:'include'}) 带 Cookie 能拿到？
       → ✅ Tier 2: cookie（最常见，evaluate 步骤内 fetch）
       → ❌ → 加上 Bearer / CSRF header 后能拿到？
              → ✅ Tier 3: header（如 Twitter ct0 + Bearer）
              → ❌ → 网站有 Pinia/Vuex Store？
                     → ✅ Tier 4: intercept（Store Action + XHR 拦截）
                     → ❌ Tier 5: ui（UI 自动化，最后手段）
```

### 各策略对比

| Tier | 策略 | 速度 | 复杂度 | 适用场景 | 实例 |
|------|------|------|--------|---------|------|
| 1 | `public` | ⚡ ~1s | 最简 | 公开 API，无需登录 | Hacker News, V2EX |
| 2 | `cookie` | 🔄 ~7s | 简单 | Cookie 认证即可 | Bilibili, Zhihu, Reddit |
| 3 | `header` | 🔄 ~7s | 中等 | 需要 CSRF token 或 Bearer | Twitter GraphQL |
| 4 | `intercept` | 🔄 ~10s | 较高 | 请求有复杂签名 | 小红书 (Pinia + XHR) |
| 5 | `ui` | 🐌 ~15s+ | 最高 | 无 API，纯 DOM 解析 | 遗留网站 |

---

## Step 3: 编写适配器

### 方式 A: YAML Pipeline（声明式，推荐）

文件路径: `src/clis/<site>/<name>.yaml`，放入即自动注册。

#### Tier 1 — 公开 API 模板

```yaml
# src/clis/v2ex/hot.yaml
site: v2ex
name: hot
description: V2EX 热门话题
domain: www.v2ex.com
strategy: public
browser: false

args:
  limit:
    type: int
    default: 20

pipeline:
  - fetch:
      url: https://www.v2ex.com/api/topics/hot.json

  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      replies: ${{ item.replies }}

  - limit: ${{ args.limit }}

columns: [rank, title, replies]
```

#### Tier 2 — Cookie 认证模板（最常用）

```yaml
# src/clis/zhihu/hot.yaml
site: zhihu
name: hot
description: 知乎热榜
domain: www.zhihu.com

pipeline:
  - navigate: https://www.zhihu.com       # 先加载页面建立 session

  - evaluate: |                            # 在浏览器内发请求，自动带 Cookie
      (async () => {
        const res = await fetch('/api/v3/feed/topstory/hot-lists/total?limit=50', {
          credentials: 'include'
        });
        const d = await res.json();
        return (d?.data || []).map(item => {
          const t = item.target || {};
          return {
            title: t.title,
            heat: item.detail_text || '',
            answers: t.answer_count,
          };
        });
      })()

  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      heat: ${{ item.heat }}
      answers: ${{ item.answers }}

  - limit: ${{ args.limit }}

columns: [rank, title, heat, answers]
```

> **关键**: `evaluate` 步骤内的 `fetch` 运行在浏览器页面内，自动携带 `credentials: 'include'`，无需手动处理 Cookie。

#### 进阶 — 带搜索参数

```yaml
# src/clis/zhihu/search.yaml
site: zhihu
name: search
description: 知乎搜索

args:
  keyword:
    type: str
    required: true
    description: Search keyword
  limit:
    type: int
    default: 10

pipeline:
  - navigate: https://www.zhihu.com

  - evaluate: |
      (async () => {
        const q = encodeURIComponent('${{ args.keyword }}');
        const res = await fetch('/api/v4/search_v3?q=' + q + '&t=general&limit=${{ args.limit }}', {
          credentials: 'include'
        });
        const d = await res.json();
        return (d?.data || [])
          .filter(item => item.type === 'search_result')
          .map(item => ({
            title: (item.object?.title || '').replace(/<[^>]+>/g, ''),
            type: item.object?.type || '',
            author: item.object?.author?.name || '',
            votes: item.object?.voteup_count || 0,
          }));
      })()

  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      type: ${{ item.type }}
      author: ${{ item.author }}
      votes: ${{ item.votes }}

  - limit: ${{ args.limit }}

columns: [rank, title, type, author, votes]
```

### 方式 B: TypeScript 适配器（编程式）

适用于需要 XHR 拦截、GraphQL、分页、复杂数据转换等场景。

文件路径: `src/clis/<site>/<name>.ts`，还需要在 `src/clis/index.ts` 中 import 注册。

#### Tier 3 — Header 认证（Twitter）

```typescript
// src/clis/twitter/search.ts
import { cli, Strategy } from '../../registry.js';

cli({
  site: 'twitter',
  name: 'search',
  description: 'Search tweets',
  strategy: Strategy.HEADER,
  args: [{ name: 'keyword', required: true }],
  columns: ['rank', 'author', 'text', 'likes'],
  func: async (page, kwargs) => {
    await page.goto('https://x.com');
    const data = await page.evaluate(`
      (async () => {
        // 从 Cookie 提取 CSRF token
        const ct0 = document.cookie.split(';')
          .map(c => c.trim())
          .find(c => c.startsWith('ct0='))?.split('=')[1];
        if (!ct0) return { error: 'Not logged in' };

        const bearer = 'AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D...';
        const headers = {
          'Authorization': 'Bearer ' + decodeURIComponent(bearer),
          'X-Csrf-Token': ct0,
          'X-Twitter-Auth-Type': 'OAuth2Session',
        };

        const variables = JSON.stringify({ rawQuery: '${kwargs.keyword}', count: 20 });
        const url = '/i/api/graphql/xxx/SearchTimeline?variables=' + encodeURIComponent(variables);
        const res = await fetch(url, { headers, credentials: 'include' });
        return await res.json();
      })()
    `);
    // ... 解析 data
  },
});
```

#### Tier 4 — Store Action + XHR 拦截（小红书）

```typescript
// src/clis/xiaohongshu/search.ts
import { cli, Strategy } from '../../registry.js';

cli({
  site: 'xiaohongshu',
  name: 'search',
  description: '搜索小红书笔记',
  strategy: Strategy.COOKIE,   // 实际是 intercept 模式
  args: [{ name: 'keyword', required: true }],
  columns: ['rank', 'title', 'author', 'likes', 'type'],
  func: async (page, kwargs) => {
    await page.goto('https://www.xiaohongshu.com');
    await page.wait(2);

    const data = await page.evaluate(`
      (async () => {
        const app = document.querySelector('#app')?.__vue_app__;
        const pinia = app?.config?.globalProperties?.$pinia;
        if (!pinia?._s) return { error: 'Page not ready' };

        const searchStore = pinia._s.get('search');
        if (!searchStore) return { error: 'Search store not found' };

        // XHR 拦截：捕获 store action 发出的请求
        let captured = null;
        const origOpen = XMLHttpRequest.prototype.open;
        const origSend = XMLHttpRequest.prototype.send;
        XMLHttpRequest.prototype.open = function(m, u) {
          this.__url = u;
          return origOpen.apply(this, arguments);
        };
        XMLHttpRequest.prototype.send = function(b) {
          if (this.__url?.includes('search/notes')) {
            const x = this;
            const orig = x.onreadystatechange;
            x.onreadystatechange = function() {
              if (x.readyState === 4 && !captured) {
                try { captured = JSON.parse(x.responseText); } catch {}
              }
              if (orig) orig.apply(this, arguments);
            };
          }
          return origSend.apply(this, arguments);
        };

        try {
          // 触发 Store Action，让网站自己签名发请求
          searchStore.mutateSearchValue('${kwargs.keyword}');
          await searchStore.loadMore();
          await new Promise(r => setTimeout(r, 800));
        } finally {
          // 恢复原始 XHR
          XMLHttpRequest.prototype.open = origOpen;
          XMLHttpRequest.prototype.send = origSend;
        }

        if (!captured?.success) return { error: captured?.msg || 'Search failed' };
        return (captured.data?.items || []).map(i => ({
          title: i.note_card?.display_title || '',
          author: i.note_card?.user?.nickname || '',
          likes: i.note_card?.interact_info?.liked_count || '0',
          type: i.note_card?.type || '',
        }));
      })()
    `);

    if (!Array.isArray(data)) return [];
    return data.slice(0, kwargs.limit || 20).map((item, i) => ({
      rank: i + 1, ...item,
    }));
  },
});
```

> **XHR 拦截核心思路**：不自己构造签名，而是劫持网站自己的 `XMLHttpRequest`，让网站的 Store Action 发出正确签名的请求，我们只是"窃听"响应。用完后必须恢复原始方法。

---

## Step 4: 测试

### 快速测试

```bash
# 公开 API（毫秒级响应）
opencli v2ex hot --limit 3

# 浏览器命令（需 Chrome + MCP Bridge）
opencli bilibili hot --limit 3

# 带参数
opencli bilibili search --keyword "rust" --limit 3
opencli zhihu search --keyword "AI" --limit 5
```

### Verbose 模式

```bash
# 查看 pipeline 每步的输入输出
opencli bilibili hot --limit 1 -v
```

输出示例：
```
  [1/4] navigate → https://www.bilibili.com
       → (no data)
  [2/4] evaluate → (async () => { const res = await fetch(…
       → [{title: "…", author: "…", play: 230835}]
  [3/4] map (rank, title, author, play, danmaku)
       → [{rank: 1, title: "…", author: "…"}]
  [4/4] limit → 1
       → [{rank: 1, title: "…"}]
```

### 输出格式验证

```bash
# 确认表格渲染正确
opencli mysite hot -f table

# 确认 JSON 可被 jq 解析
opencli mysite hot -f json | jq '.[0]'

# 确认 CSV 可被导入
opencli mysite hot -f csv > data.csv
```

---

## Step 5: 注册 & 发布

### YAML 适配器

放入 `src/clis/<site>/<name>.yaml` 即自动注册，无需额外操作。

### TS 适配器

在 `src/clis/index.ts` 添加 import：

```typescript
import './mysite/search.js';
```

### 验证注册

```bash
opencli list                     # 确认新命令出现
opencli validate mysite          # 校验定义完整性
```

### 提交

```bash
git add src/clis/mysite/
git commit -m "feat(mysite): add hot and search adapters"
git push
```

---

## 常见陷阱

| 陷阱 | 表现 | 解决方案 |
|------|------|---------|
| 缺少 `navigate` | evaluate 报 `Target page context` 错误 | 在 evaluate 前加 `navigate:` 步骤 |
| 嵌套字段访问 | `${{ item.node?.title }}` 不工作 | 在 evaluate 中 flatten 数据，不在模板中用 optional chaining |
| 缺少 `strategy: public` | 公开 API 也启动浏览器，7s → 1s | 公开 API 加上 `strategy: public` + `browser: false` |
| evaluate 返回字符串 | map 步骤收到 `""` 而非数组 | pipeline 有 auto-parse，但建议在 evaluate 内 `.map()` 整形 |
| 搜索参数被 URL 编码 | `${{ args.keyword }}` 被浏览器二次编码 | 在 evaluate 内用 `encodeURIComponent()` 手动编码 |
| Cookie 过期 | 返回 401 / 空数据 | 在浏览器里重新登录目标站点 |
| Extension tab 残留 | Chrome 多出 `chrome-extension://` tab | 已自动清理；若残留，手动关闭即可 |

---

## 用 AI Agent 自动生成适配器

最快的方式是让 AI Agent 完成全流程：

```bash
# 一键：探索 → 分析 → 合成 → 注册
opencli generate https://www.example.com --goal "hot"

# 或分步执行：
opencli explore https://www.example.com --site mysite   # 发现 API
opencli synthesize mysite                                # 生成候选 YAML
opencli verify mysite/hot --smoke                        # 冒烟测试
```

生成的候选 YAML 保存在 `.opencli/explore/mysite/candidates/`，可直接复制到 `src/clis/mysite/` 并微调。
