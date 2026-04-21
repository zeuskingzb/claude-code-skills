---
name: pinme-deploy
description: 使用 PinMe（零配置、无服务器、基于 IPFS 的前端静态网站一键部署工具）将静态页面/前端项目一键发布到 IPFS。**只要用户发出"部署这个页面 / 上传到 IPFS / 用 pinme 发布 / 把这个项目发上去 / 发布静态站点 / pinme upload / 一键部署前端 / 把 dist 发布出去 / 生成一个可访问的链接"等类似诉求，或希望将本地前端产物（dist/build/out/纯 HTML 目录）快速发布成可公网访问的静态站点时，都应主动触发本 skill**。本 skill 负责环境检查、按框架选择构建命令、识别正确的静态输出目录、执行 pinme upload 并回传访问链接，同时处理路由模式、目录规则、文件大小等常见陷阱。不适用于动态服务端渲染（SSR）部署、后端服务部署、需要数据库/API 的全栈应用部署。
---

# PinMe Deploy（IPFS 静态站点一键发布）

本 skill 的目标：帮助用户用最少的命令，把本地前端静态产物通过 PinMe 发布到 IPFS，并拿到可访问链接。

## 核心哲学

> **好发布 = 对的目录 + 对的路由模式 + 干净的上传内容。**

- 不上传源码（src/、node_modules/、.git/、.env）。
- 不在未确认入口文件（index.html）存在的前提下执行 upload。
- 不对非静态项目（SSR / 需要运行时服务端）硬塞给 pinme。

## 工作流程

### Step 1：环境检查

**必须先执行**：

```bash
pinme -v
```

- 如果输出版本号 → 环境 OK，继续 Step 2。
- 如果报 `command not found` / 无输出 → 指导用户安装：

```bash
npm install -g pinme
```

安装完成后再次执行 `pinme -v` 确认。

### Step 2：识别项目类型，决定是否需要构建

先查看项目根目录结构（`package.json`、`vite.config.*`、`next.config.*`、`vue.config.*`、`index.html` 等），对照下表判断：

| 项目类型 | 判定特征 | 构建命令 | 输出目录 |
|---------|---------|---------|---------|
| Vite | `vite.config.{js,ts}` | `npm run build` | `dist/` |
| Create React App | `react-scripts` in deps | `npm run build` | `build/` |
| Next.js（静态导出） | `next.config.*` + `output: 'export'` 或 `next export` | `npm run build && npm run export` | `out/` |
| Vue CLI | `@vue/cli-service` in deps | `npm run build` | `dist/` |
| 纯 HTML | 根目录直接有 `index.html` | **无需构建** | 项目根目录 |
| 其它/未知 | - | 询问用户对应的构建命令与输出目录 | - |

**执行构建**（仅当需要时）：在项目根目录执行对应构建命令，完成后确认输出目录存在且包含 `index.html`。

### Step 3：上传发布

```bash
pinme upload <静态目录>
```

例：
- Vite / Vue CLI: `pinme upload dist`
- CRA: `pinme upload build`
- Next.js 静态: `pinme upload out`
- 纯 HTML: `pinme upload .`（在项目根目录）

将命令输出中的**访问链接/CID** 回传给用户。

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `pinme upload <目录>` | 上传静态目录 |
| `pinme list` | 查看历史上传记录 |
| `pinme rm <hash>` | 删除已发布内容 |
| `pinme bind <目录> --domain <域名>` | 绑定自定义域名（需 VIP） |
| `pinme my-domains` | 查看已绑定域名 |
| `pinme set-appkey <AppKey>` | 设置登录密钥 |

## 目录规则与限制（上传前务必核对）

- **必须**包含 `index.html`（否则 pinme 直接报错）。
- **禁止**上传：`node_modules/`、`.git/`、`.env`、`src/`、各类配置文件（如 `vite.config.ts`、`tsconfig.json`）。
- 单文件 ≤ 200MB，总目录 ≤ 1GB。
- 上传前应确认目标目录只包含编译后的静态资源（html / js / css / 图片 / 字体 / favicon 等）。

## 路由模式（前端项目必读）⚠️

IPFS 网关不支持服务端重写规则，**必须使用 hash 路由**，否则刷新子页面必 404。

- React: 使用 `HashRouter` 替代 `BrowserRouter`。
- Vue: 使用 `createWebHashHistory()` 替代 `createWebHistory()`。
- 其它框架：一律改为 hash 模式 / 静态预渲染。

如果用户项目当前是 history 模式，**在上传前提醒用户切换**；若用户仍坚持，需告知子页面刷新会 404。

## 常见错误与处置

| 错误提示 | 可能原因 | 解决方案 |
|---------|---------|---------|
| `directory not found` | 路径拼错 / 未构建 | 核对路径，确认构建产物目录已生成 |
| `index.html not found` | 上传的不是构建产物目录 | 改用正确的输出目录（dist/build/out） |
| `upload failed` | 网络波动 / 认证失效 | 检查网络；必要时 `pinme set-appkey <AppKey>` 重新登录 |
| `file too large` | 单文件 >200MB 或总量 >1GB | 压缩图片/视频/字体；按需拆分；启用打包分片 |
| 子页面刷新 404 | 使用了 history 路由 | 改为 hash 路由后重新构建并上传 |

## 执行清单（Checklist）

每次发布前默认按此顺序核查：

1. `pinme -v` 有版本号输出。
2. 已识别项目类型，构建命令与输出目录匹配。
3. 构建完成（若需要），输出目录存在。
4. 输出目录根下有 `index.html`。
5. 目录大小合规（单文件 ≤200MB，总量 ≤1GB）。
6. 前端路由为 hash 模式（如适用）。
7. 执行 `pinme upload <目录>`，拿到 CID / 访问链接回传用户。

## 与用户的交互风格

- 每一步先**说一句你要做什么**，再执行。
- 非必要不追问；项目类型/构建方式**不明确时**才问，一次问清楚（构建命令？输出目录？是否已构建？）。
- 上传成功后，直接把**访问链接**显眼地贴出来，并顺手给出 `pinme list` / `pinme rm <hash>` 的后续操作提示。
