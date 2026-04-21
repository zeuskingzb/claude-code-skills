
创建pinme-deploy skill，该skill利用pinme来实现静态页面发布功能。

介绍：PinMe 是零配置、无服务器、基于 IPFS 的前端静态网站一键部署工具。

### Setp 1: 检查环境
使用`pinme -v`指令，通过是否输出版本号，来判断当前有无安装pinme
pinme安装命令npm install -g pinme


### Setp 2：构建并上传
- 构建（可选）：若非原生前端项目，请执行对应的前端项目构建命令
- 上传：pinme upload <静态目录>

## 常用命令

| 命令 | 说明 |
|------|------|
| pinme upload <目录> | 上传静态目录 |
| pinme list | 查看上传记录 |
| pinme rm <hash> | 删除已发布内容 |
| pinme bind <目录> --domain <域名> | 绑定域名（需 VIP） |
| pinme my-domains | 查看已绑定域名 |
| pinme set-appkey <AppKey> | 设置登录密钥 |

## 框架构建目录对照

| 框架 | 构建命令 | 输出目录 |
|------|----------|----------|
| Vite | npm run build | dist/ |
| Create React App | npm run build | build/ |
| Next.js (static) | npm run build && npm run export | out/ |
| Vue CLI | npm run build | dist/ |
| 纯 HTML | 无需构建 | 项目根目录 |

## 目录规则与限制

- 必须包含 index.html
- 禁止上传：node_modules、.git、.env、src、配置文件
- 单文件 ≤ 200MB，总目录 ≤ 1GB

## 路由注意事项

⚠️ 必须使用 hash 模式路由：
- React: HashRouter
- Vue: createWebHashHistory

## 常见错误处理

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| directory not found | 目录路径错误 | 检查路径 |
| index.html not found | 缺少入口文件 | 确保 index.html 存在 |
| upload failed | 网络或认证问题 | 检查网络和 appkey |
| file too large | 超过限制 | 压缩资源 |
| 子页面 404 | history 路由 | 改用 hash 模式 |

请将skill保存到当前目录.claude/skill下