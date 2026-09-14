# Cloudflare Workers 部署指南

本项目（`sub-web`）基于 Vue 3 + Vite 单页应用（SPA）架构构建，已通过 Cloudflare 官方最新的 **Workers Static Assets（静态资产原生托管）** 进行适配，可直接在 Cloudflare 全球边缘网络（Workers）上高效、低成本地部署与运行。

---

## 🌟 核心特性与优势

1. **原生静态资产托管 (Workers Static Assets)**：
   - 前端构建产物（HTML、CSS、JS、静态资源）直接上传至 Cloudflare 边缘存储，由全球 Anycast CDN 高速分发与分层缓存。
   - 静态资产访问直接由 CDN 命中返回，**不消耗 Worker 计算调用量（Billable Invocations）**，免费额度友好。

2. **开箱即用的 SPA 路由回退**：
   - 配置文件中设置 `assets.not_found_handling = "single-page-application"`。
   - 当用户在浏览器直接访问或刷新子路由时，Cloudflare 边缘节点自动回退返回 `index.html`（状态码 200），完美兼容 Vue Router 的 HTML5 History 模式（`createWebHistory`）。

3. **极简运维与持续集成**：
   - 仅需一个标准的 `wrangler.jsonc`，无需维护复杂的 Nginx 反向代理或 Docker 容器。
   - 可与 GitHub Actions、Cloudflare Workers CI/CD 轻松集成。

---

## 📦 环境准备

- **Node.js**: `>= 22.x`（推荐与项目一致使用 `24.x`；Wrangler v4 要求 Node.js 22 或更高版本）
- **Yarn**: `1.22+`
- **Cloudflare 账户**: [注册或登录 Cloudflare](https://dash.cloudflare.com/)

---

## 🚀 快速上手

### 1. 登录 Cloudflare 账户

首次在本地使用 Wrangler 部署前，需完成身份认证：

```bash
npx wrangler login
```

该命令会自动打开浏览器进行 OAuth 授权。授权完成后，Wrangler 将在本地保存访问凭证。

> 💡 **CI/CD 环境提示**：在自动化流水线中，可通过环境变量 `CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_ACCOUNT_ID` 进行免交互认证。

### 2. 调试与本地预览

在本地构建并使用 Wrangler 本地运行态预览 SPA：

```bash
# 构建生产产物并使用 Wrangler 启动本地边缘模拟服务
yarn preview:worker
```

服务启动后，可在控制台输出的本地地址（通常为 `http://localhost:8787`）进行访问和功能测试。

### 3. 一键部署到 Cloudflare Workers

执行以下命令即可自动完成前端打包与边缘部署：

```bash
yarn deploy
```

部署完成后，控制台将输出分配的 Worker 访问地址（例如 `https://sub-web.<你的子域名>.workers.dev`）。

---

## ⚙️ 配置文件解析 (`wrangler.jsonc`)

项目根目录的 `wrangler.jsonc` 包含部署至 Cloudflare Workers 的核心配置：

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "sub-web",                            // Worker 服务名称，可在 Cloudflare 控制台展示
  "compatibility_date": "2026-09-14",          // 兼容性日期，启用最新的边缘运行时特性
  "assets": {
    "directory": "./dist",                      // 指定 Vite 构建产物目录
    "not_found_handling": "single-page-application" // 启用 SPA 路由自动回退至 index.html
  }
}
```

---

## 🌐 自定义域名与路由绑定

默认情况下，Cloudflare 为每个 Worker 分配 `*.workers.dev` 免费子域名。如需绑定自己的独立域名：

### 方法 1：通过 Cloudflare 控制台绑定（推荐）
1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)。
2. 进入 **Compute (Workers & Pages)** -> 选择 **sub-web**。
3. 进入 **Settings** -> **Domains & Routes**。
4. 点击 **Add** -> 选择 **Custom Domain**，输入你已在 Cloudflare 解析的域名（如 `sub.example.com`），保存后系统将自动签发 SSL 证书并生效。

### 方法 2：在 `wrangler.jsonc` 中声明
可直接在 `wrangler.jsonc` 中增加 `routes` 配置：

```jsonc
{
  "name": "sub-web",
  "compatibility_date": "2026-09-14",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  },
  "routes": [
    { "pattern": "sub.example.com/*", "custom_domain": true }
  ]
}
```

---

## 🔧 环境变量与后端地址定制

`sub-web` 的订阅转换后端（`VITE_SUBCONVERTER_DEFAULT_BACKEND`）等配置属于前端编译期注入（Vite `import.meta.env`）。

如需调整默认后端、短链接 API 或页面标题：
1. 修改项目根目录的 `.env` 文件（或创建 `.env.production`）。
2. 重新执行 `yarn deploy`。

示例配置：
```env
# 指定你的专属转换后端地址
VITE_SUBCONVERTER_DEFAULT_BACKEND="https://your-backend.domain.com"
VITE_PROJECT="https://github.com/CareyWang/sub-web"
```

---

## 🧩 进阶：扩展 Worker 接口反代（解决跨域 CORS）

如果在调用某些私有或第三方 Subconverter 后端时出现浏览器跨域（CORS）报错，可以利用 Cloudflare Workers 的边缘脚本（Compute）能力作为反向代理。

### 扩展步骤：

1. 在项目根目录或 `worker/` 下创建入口脚本 `worker/index.js`：

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url)

    // 拦截 /api/sub 请求并代理转发至实际后端
    if (url.pathname.startsWith('/api/sub')) {
      const targetUrl = new URL(url.searchParams.get('url') || 'https://api.wcc.best/sub' + url.search)
      const modifiedRequest = new Request(targetUrl, {
        method: request.method,
        headers: request.headers
      })
      const response = await fetch(modifiedRequest)
      const newHeaders = new Headers(response.headers)
      newHeaders.set('Access-Control-Allow-Origin', '*')
      return new Response(response.body, {
        status: response.status,
        headers: newHeaders
      })
    }

    // 其它非 API 请求回退给静态资产
    return env.ASSETS.fetch(request)
  }
}
```

2. 修改 `wrangler.jsonc`，启用 Worker 脚本与定向路由：

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "sub-web",
  "compatibility_date": "2026-09-14",
  "main": "worker/index.js",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*"]
  }
}
```
这样既保留了静态资源完全由边缘 CDN 零成本分发的特性，又可在需要时利用 Worker 动态处理 API。
