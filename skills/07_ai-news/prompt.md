使用skill-creator 创建一个AI资讯skill，用于获取每日最新的AI资讯

## Step 1：数据获取
使用agent-browser skill分别访问以下两个数据源：
1. AIBase — `https://www.aibase.com/zh/news`
2. 36氪 AI 频道 — `https://www.36kr.com/information/AI/`

将抓取到的数据整理为以下格式，保存为 `ai-news-data.js`：
```js
const NEWS_DATA = [
  {
    "title": "文章标题",
    "description": "文章主旨80字以内的精炼摘要",
    "link": "文章详情链接",
    "tags": ["AI", "相关标签1", "相关标签2"]
  }
];
```

**注意事项**
- JS 文件中字符串内部的双引号必须在前面加反斜杠 `\` 转义
- 如果某个网站无法访问或加载失败，跳过该数据源，继续使用其他可用数据源
- 每个数据源至少抓取 5 条，两个数据源合计目标 15-20 条
- 去除明显重复的资讯（相同主题、相似标题）
- 如果链接不完整（如缺少域名前缀），自动补全
- tags 数组包含 2-4 个相关标签，根据标题和内容自动推断
- description 控制在 80 字以内

## Step 2：展示页面
创建赛博朋克风格的 HTML 展示页面，**通过 `<script src>` 引入数据文件**，实现数据与模板分离：

```html
<!-- 在 </body> 前引入数据文件和渲染脚本 -->
<script src="ai-news-data.js"></script>
<script>
// NEWS_DATA 变量来自 ai-news-data.js
// 此处编写渲染逻辑，将 NEWS_DATA 渲染到页面
</script>
```

**HTML 页面要求：**
- 赛博朋克视觉风格（霓虹灯效果、深色背景、科技感）
- 卡片式布局展示每条资讯
- 显示标题、描述、标签、原文链接
- 响应式设计，支持移动端和桌面端
- 包含日期和资讯条数信息

**交付文件：**
1. `ai-news-data.js` — 数据文件（JS 变量）
2. `ai-news.html` — 展示页面（通过 script 标签引入数据）

请将该skill保存到当前工作目录 .claude/skills 下
