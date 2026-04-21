---
name: ai-news
description: 每日 AI 资讯聚合 skill：自动从 AIBase、36 氪 AI 频道等主流 AI 资讯站点抓取当天最新动态，去重、补全、打标签后整理为标准化数据文件 `ai-news-data.js`，并生成一份赛博朋克（Cyberpunk）视觉风格、霓虹光效、卡片式、响应式的 `ai-news.html` 展示页面（通过 `<script src>` 实现数据与模板分离）。**只要用户发出"今日 AI 资讯 / 每日 AI 新闻 / 抓一下 AI 动态 / 最新 AI news / AI 行业资讯 / AI 日报 / 爬一下 AIBase / 看看 36 氪 AI / 整理一下 AI 新闻 / 生成 AI 资讯页 / 做个 AI 新闻看板"等类似诉求，或希望把多源 AI 资讯汇总成一个可浏览页面时，都应主动触发本 skill**。本 skill 专注于"数据抓取 → 标准化 → 模板化展示"的完整闭环，强调数据与 HTML 模板严格分离、字符串转义规范、卡片式赛博朋克 UI。不适用于非 AI 类资讯抓取、需要登录鉴权的数据源、实时推送/订阅服务、纯 RSS 订阅器、后端爬虫服务部署。
---

# AI 资讯聚合（AI News Daily）

本 skill 的目标：当用户想看"今天 AI 圈发生了什么"时，从主流 AI 资讯源抓取当日动态，产出一份**可直接打开浏览**的赛博朋克风格资讯看板。

## 核心哲学

> **好资讯聚合 = 多源抓取 + 干净去重 + 数据模板分离 + 能跑能看的可视化。**

- 不做"复制粘贴"——数据源多于 1 个，去重后合并。
- 不做"数据混在 HTML 里"——数据 (`ai-news-data.js`) 与模板 (`ai-news.html`) 必须分离，通过 `<script src>` 引入。
- 不做"字符串炸弹"——JS 字符串内的双引号必须用 `\"` 转义，换行用 `\n`，禁止生成可能 SyntaxError 的数据文件。
- 不做"空看板"——任一数据源挂了要降级继续，绝不交付空白页面。
- 不做"Lorem Ipsum"——标题/描述必须是真实抓取内容；描述若过长，精炼到 80 字以内，但**不得编造**。

## 工作流程（强制 3 步，不得跳步）

### Step 1：数据抓取（多源并行 + 降级策略）

**数据源（主要）：**

| 来源 | URL | 备注 |
|------|-----|------|
| AIBase | `https://www.aibase.com/zh/news` | 中文 AI 垂直资讯站，更新快 |
| 36 氪 AI 频道 | `https://www.36kr.com/information/AI/` | 大厂动态 / 融资 / 产品发布为主 |

**抓取方式优先级：**

1. **首选**：调用 `agent-browser` skill（若当前环境已注册），它能处理动态渲染页面。
2. **次选**：若无 `agent-browser`，使用 `WebFetch` 工具逐个拉取 HTML，再用结构化提示词让模型自己抽取条目字段。
3. **兜底**：若 `WebFetch` 也被拒或超时，用 `WebSearch` 以 `site:aibase.com/zh/news`、`site:36kr.com AI` 搜索近 24 小时条目并提取。

**抓取字段（每条必填）：**

- `title`：原始标题，保留中文标点，**禁止**改写。
- `description`：80 字以内摘要。若原文描述超过 80 字，**总结精炼**而非截断；若原文无描述，用标题扩写一句客观概述，不得编造事实或数字。
- `link`：文章详情页链接。若是相对路径（如 `/news/123`），自动补全为 `https://www.aibase.com/zh/news/123` 或 `https://www.36kr.com/news/123` 等完整 URL。
- `tags`：2–4 个相关标签（数组）。根据标题/描述推断，优先从下列词表取词，保持简洁：
  - 通用：`AI`、`大模型`、`LLM`、`AIGC`、`多模态`
  - 角色：`OpenAI`、`Anthropic`、`Google`、`Meta`、`字节`、`百度`、`阿里`、`腾讯`
  - 场景：`融资`、`发布`、`开源`、`收购`、`研究`、`监管`、`安全`、`Agent`、`编程`、`视频生成`、`图像生成`、`语音`、`机器人`
  - 第一个 tag 建议固定为 `AI`，方便页面筛选。

**数量要求：**

- 每个可用数据源至少 5 条，两源合计目标 **15–20 条**。
- 任一数据源失败（访问超时 / 结构变更 / 被风控），**跳过该源继续**，不要中断整个流程。两源全挂时才向用户报错。

**去重规则：**

- 标题完全相同 → 保留先抓到的一条。
- 标题高度相似（如只差语气词 / 标点 / 是否带书名号）→ 视为重复，保留更完整的一条。
- 同一主题但两源表述不同，视为有效的交叉印证，**保留**。

**产物 1：`ai-news-data.js`**

严格按以下格式写入（**双引号转义、无尾逗号、UTF-8**）：

```js
const NEWS_DATA = [
  {
    "title": "文章标题",
    "description": "文章主旨 80 字以内的精炼摘要",
    "link": "https://www.aibase.com/zh/news/xxx",
    "tags": ["AI", "大模型", "OpenAI"]
  },
  {
    "title": "另一条标题",
    "description": "另一条摘要",
    "link": "https://www.36kr.com/p/xxx",
    "tags": ["AI", "融资", "Agent"]
  }
];
```

**转义清单（生成前自检）：**

- 字符串内的 `"` → `\"`
- 字符串内的 `\` → `\\`
- 字符串内的换行 → `\n`（一般直接去掉换行，用空格替代）
- 不得出现裸的单行 JS 注释 `//` 落在 URL 里被截断（URL 请勿换行）
- 数组/对象**末尾不加逗号**

### Step 2：生成赛博朋克展示页 `ai-news.html`

**严格要求：数据与模板分离。** 在 `</body>` 前：

```html
<script src="ai-news-data.js"></script>
<script>
  // NEWS_DATA 全局变量来自 ai-news-data.js
  // 此处只负责把 NEWS_DATA 渲染到 DOM
</script>
```

**不允许**把 `NEWS_DATA` 写死在 `ai-news.html` 内部。

#### 视觉规范（赛博朋克 Cyberpunk）

| 项目 | 规则 |
|------|------|
| 背景 | 深色主背景（`#0a0a14` / `#05050d`），可叠加微弱的扫描线 / 网格纹理 |
| 主色（霓虹） | 青色 `#00f0ff`（Neon Cyan） |
| 辅色（霓虹） | 品红 `#ff00d4`（Neon Magenta）、紫 `#a100ff` |
| 点缀色 | 柠檬黄 `#f7ff00`（用于关键数字 / hover） |
| 字体 | 标题使用等宽或几何感字体（如 `Orbitron`、`JetBrains Mono`、`'Courier New'`），正文使用中文无衬线（`'PingFang SC'`、`'Microsoft YaHei'`, sans-serif） |
| 光效 | 文字/边框使用 `text-shadow` / `box-shadow` 叠加霓虹光晕，`filter: drop-shadow()` 增强 |
| 装饰 | 允许少量扫描线动画、故障（glitch）文字、网格背景、霓虹分割线 |

**禁止：** 纯白背景、圆润扁平化风格、Material Design 配色。这是赛博朋克，不是企业官网。

#### 布局与组件

- **顶部 Header**
  - 站名：`AI NEWS // DAILY`（霓虹标题，带 glitch 效果）
  - 副标题：`Cyber Intelligence Feed`
  - 当天日期（动态从 `new Date()` 取，格式：`YYYY-MM-DD` 或 `YYYY.MM.DD`）
  - 资讯条数：`// ${NEWS_DATA.length} ENTRIES`

- **资讯卡片网格**
  - 桌面端：3 列 Grid；平板：2 列；移动端：1 列。
  - 单张卡片包含：
    - 序号徽章（`#01`、`#02`…）带霓虹边框
    - 标题（霓虹发光，hover 时增强光晕）
    - 描述（正文颜色略暗 `#c0c0d0`）
    - 标签（小胶囊，透明底 + 霓虹描边，第一个标签使用青色系，其余用品红/紫色系轮换）
    - "READ MORE →" 链接（青色，hover 变品红并下划线）
  - 卡片背景：`rgba(20, 20, 40, 0.6)` 半透明 + `backdrop-filter: blur(6px)`
  - 卡片边框：`1px solid` 霓虹色，hover 时边框变色 + `box-shadow` 外发光

- **底部 Footer**
  - 数据源列表（AIBase、36Kr 等文字小胶囊）
  - 版权 / 生成时间戳

#### 响应式

- `max-width: 1200px` 容器居中
- 断点：`>1024px` 3 列；`>640px` 2 列；其余 1 列
- 卡片之间 `gap: 20px–24px`

#### 渲染脚本骨架（示例）

```html
<script src="ai-news-data.js"></script>
<script>
  (function render() {
    if (typeof NEWS_DATA === 'undefined' || !Array.isArray(NEWS_DATA)) {
      document.getElementById('grid').innerHTML =
        '<div class="error">// DATA FEED OFFLINE</div>';
      return;
    }
    const grid = document.getElementById('grid');
    grid.innerHTML = NEWS_DATA.map((item, i) => {
      const idx = String(i + 1).padStart(2, '0');
      const tags = (item.tags || [])
        .map((t, ti) => `<span class="tag tag-${ti % 3}">${t}</span>`)
        .join('');
      return `
        <article class="card">
          <div class="idx">#${idx}</div>
          <h2 class="title">${item.title}</h2>
          <p class="desc">${item.description}</p>
          <div class="tags">${tags}</div>
          <a class="more" href="${item.link}" target="_blank" rel="noopener">READ MORE →</a>
        </article>
      `;
    }).join('');
    document.getElementById('count').textContent = NEWS_DATA.length;
    document.getElementById('date').textContent =
      new Date().toISOString().slice(0, 10).replace(/-/g, '.');
  })();
</script>
```

#### 样式要点（CSS 片段参考）

```css
:root {
  --bg: #05050d;
  --bg-card: rgba(20, 20, 40, 0.6);
  --neon-cyan: #00f0ff;
  --neon-magenta: #ff00d4;
  --neon-purple: #a100ff;
  --neon-yellow: #f7ff00;
  --text: #e0e0f0;
  --text-dim: #8a8aa0;
}
body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  background-image:
    linear-gradient(rgba(0, 240, 255, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 240, 255, 0.03) 1px, transparent 1px);
  background-size: 40px 40px;
  min-height: 100vh;
}
.title {
  color: var(--neon-cyan);
  text-shadow: 0 0 6px var(--neon-cyan), 0 0 14px rgba(0, 240, 255, 0.5);
  font-family: 'Orbitron', 'JetBrains Mono', monospace;
}
.card {
  background: var(--bg-card);
  border: 1px solid rgba(0, 240, 255, 0.4);
  backdrop-filter: blur(6px);
  transition: all 0.25s;
  padding: 18px;
  border-radius: 4px;
  clip-path: polygon(0 0, 100% 0, 100% calc(100% - 14px), calc(100% - 14px) 100%, 0 100%);
}
.card:hover {
  border-color: var(--neon-magenta);
  box-shadow: 0 0 18px rgba(255, 0, 212, 0.4);
  transform: translateY(-2px);
}
.tag {
  display: inline-block;
  padding: 2px 10px;
  margin: 2px 4px 2px 0;
  font-size: 12px;
  border: 1px solid currentColor;
  border-radius: 2px;
  letter-spacing: 0.5px;
}
.tag-0 { color: var(--neon-cyan); }
.tag-1 { color: var(--neon-magenta); }
.tag-2 { color: var(--neon-purple); }
.more {
  color: var(--neon-cyan);
  text-decoration: none;
  letter-spacing: 1px;
  font-family: 'JetBrains Mono', monospace;
}
.more:hover { color: var(--neon-magenta); text-shadow: 0 0 8px currentColor; }
```

### Step 3：产物交付与自检

**交付文件（位置：当前工作目录根下，两者必须在同一目录）：**

1. `ai-news-data.js` — 数据文件
2. `ai-news.html` — 展示页面

**自检清单（交付前逐项过）：**

1. `ai-news-data.js` 能用 `node -e "require('./ai-news-data.js')"` 级别语法检查（至少确认括号、引号、逗号成对闭合，无尾逗号）。
2. `ai-news-data.js` 里不包含未转义的 `"`、换行、反斜杠。
3. `ai-news.html` 中**没有**硬编码的资讯数据，只通过 `<script src="ai-news-data.js"></script>` 引入。
4. 用浏览器打开 `ai-news.html` 时：
   - 卡片数等于 `NEWS_DATA.length`。
   - 日期显示为今天。
   - 所有 `link` 可点击且 `target="_blank"`。
   - 桌面端至少 3 列，窄屏自动坍缩。
5. 数据条数 ≥ 单源 5 条、双源合计 15–20 条（若某源失败且已降级，向用户说明原因）。

**交付时向用户汇报：**

- 文件路径（`./ai-news-data.js`、`./ai-news.html`）
- 抓取条数与来源分布（如：AIBase 8 条 + 36Kr 7 条，去重 1 条，最终 14 条）
- 若有数据源降级，明确说明哪一源失败、用了什么兜底
- 建议：可直接双击打开 `ai-news.html`；若想分享，可用 `pinme-deploy` skill 一键部署到 IPFS。

## 常见错误与处置

| 错误提示 | 可能原因 | 解决方案 |
|---------|---------|---------|
| `Uncaught SyntaxError: Unexpected token` | JS 字符串中未转义的 `"` 或换行 | 重新生成 `ai-news-data.js`，对字段做转义（`replace('"','\\"')`） |
| 页面空白 | `ai-news-data.js` 未同级放置 / 路径错 | 确认两文件在同一目录；`<script src>` 用相对路径 |
| 卡片挤成一团 | 忘记写 media query | 加 `@media (max-width: 1024px)` / `(max-width: 640px)` |
| 链接无法点击 | `link` 字段是相对路径未补全 | 回到 Step 1 补全域名 |
| 某源 0 条 | 该站结构变了 / 风控 | 标注降级，使用 WebSearch 兜底，或只用另一源 |
| 字体没生效 | `Orbitron` 等未引入 | 在 `<head>` 用 Google Fonts `<link>` 引入，或降级为 `'Courier New', monospace` |
| 中文乱码 | 文件未用 UTF-8 保存 | 写入时强制 UTF-8，`<meta charset="utf-8">` 放 `<head>` 第一行 |

## 与用户的交互风格

- 每一步先**说一句你在做什么**（"正在抓 AIBase…"、"36Kr 返回 403，降级用 WebSearch"），再动手。
- 抓取阶段**不要**追加问题打扰用户，除非两个源都失败。
- 产物交付时，把**文件路径 + 条数 + 来源分布**一次性贴出，不要拆成多轮对话。
- 用户若追加"再多一点"/"帮我部署"等需求，才继续：多抓 → 调用 `pinme-deploy` skill。

## 执行清单（Checklist）

每次执行默认按此顺序核查：

1. 两个数据源都已尝试（AIBase + 36Kr），记录成功/失败。
2. 抓取到条目数 ≥ 单源 5、合计 15–20（降级情形除外）。
3. 字段齐全：title / description（≤80 字）/ link（完整 URL）/ tags（2–4 个）。
4. 已去重，标题完全重复与高度相似均已处理。
5. `ai-news-data.js` 生成，字符串转义正确，无尾逗号。
6. `ai-news.html` 生成，采用 `<script src>` 引入数据，未硬编码。
7. 赛博朋克视觉到位：深色背景 + 霓虹色 + 卡片玻璃拟态 + 响应式布局。
8. 浏览器打开自测：渲染正常、链接可跳转、日期为今天、移动端单列。
9. 向用户汇报产物路径、条数、来源分布、可选的后续（部署/扩充）。
