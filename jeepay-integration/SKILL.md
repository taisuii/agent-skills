---
name: logto-integration
description: Guide agents to integrate Logto (OIDC) authentication across multiple languages and frameworks. Use when the user wants to add or modify Logto-based login, logout, and token validation flows in web or backend applications.
---

# Logto 集成 Skill（多语言 / 多框架）

本 Skill 指导代理在各种语言和框架中集成 Logto 作为 OIDC 身份提供方，并参考本项目的 Next.js 集成方式（`src/libraries/logto.ts` + `src/pages/api/logto/[action].ts`）。适用于：新增登录 / 登出、保护接口、访问用户信息、在后端校验 Access Token 等场景。

## 使用前提

- 用户已经在 Logto Console 中创建了应用（Web / SPA / API 等），并拿到了：
  - `endpoint`（Logto 服务地址，如 `https://login.example.com`）
  - `appId`
  - `appSecret`（传统 Web / 后端应用）
  - 可选：API 资源标识、额外 scopes
- 用户可以配置环境变量或安全配置文件。

## 核心概念 & 配置字段

所有语言和框架的集成，核心都围绕以下配置展开：

- **endpoint**：Logto 服务地址，对应 OIDC Discovery (`.well-known/openid-configuration`)
- **appId**：Logto 中注册的应用 ID
- **appSecret**：传统 Web / 机密客户端使用的客户端密钥（SPA 通常不需要）
- **redirectUri**：登录完成后的回调地址
- **postLogoutRedirectUri / signOutRedirectUri**：登出后的跳转地址
- **scopes**：OIDC Scopes，如 `openid profile email phone custom_data`
- **resources**：API 资源标识，用于获取特定资源的 Access Token
- **cookie / storage**：前端或后端保存会话信息的方式（浏览器 Cookie、本地存储、内存等）

在本项目中，Next.js Pages Router 使用了 `@logto/next`：

- `src/libraries/logto.ts` 中创建 `logtoClient`，配置 `appId`、`appSecret`、`endpoint`、`baseUrl`、`cookieSecret`、`cookieSecure`、`scopes` 等。
- `src/pages/api/logto/[action].ts` 中：
  - 对 `/api/logto/sign-in` 注入 `ui_locales` 并指定 `redirectUri`
  - 对 `/api/logto/sign-out` 使用 `handleSignOut(baseUrl)`
  - 其他路由统一交给 `handleAuthRoutes` 处理

## 通用集成流程（各语言通用）

无论什么语言或框架，按以下步骤操作：

1. **在 Logto Console 中创建应用**
   - 选择合适类型（SPA、传统 Web、Native、Machine-to-Machine 等）
   - 配置 **回调地址**（如 `https://your-app.com/callback`、`https://your-app.com/api/logto/sign-in-callback`）
   - 配置 **登出回调地址**（如 `https://your-app.com/`）
   - 记录 `endpoint`、`appId`、`appSecret`。
2. **在应用中配置环境变量 / 配置文件**
   - 将上面的字段写入 `.env`、`appsettings.json`、YAML 等配置。
3. **引入对应语言的 Logto SDK 或 OIDC 客户端**
   - JavaScript / TypeScript：使用官方 Logto SDK（如 `@logto/next`、`@logto/browser`、`@logto/node`）或 OIDC 客户端。
   - Python：使用 OIDC 客户端库（如 `authlib`）或 Logto 提供的示例。
   - Java：使用 Spring Security OIDC 或官方 SDK。
   - Go：使用 Logto Go SDK。
   - .NET：使用 ASP.NET Core OIDC 中间件或 Logto 示例。
4. **实现登录（Sign-in）**
   - 构造 Logto 授权 URL（带 PKCE / state）。
   - 跳转到 Logto 登录页。
5. **实现回调处理（Callback）**
   - 在回调路由中：
     - 校验 `state` 和 PKCE
     - 用授权码换取 Token（Access Token、ID Token、Refresh Token）
     - 验证 ID Token（签名 / 过期时间 / audience 等）
     - 保存会话（Cookie / Session / Token 存储）。
6. **实现登出（Sign-out）**
   - 调用 Logto 的登出端点或 SDK 的登出方法
   - 清空本地会话
   - 跳转到指定的登出回调地址。
7. **保护受保护资源**
   - 在中间件 / Filter / Guard 中校验当前请求是否已登录
   - 对后端 API：验证 Bearer Token 是否合法并具有正确的 scopes / resources。

## 环境变量建议（可参考本项目）

在大多数后端 / 全栈项目中，推荐如下变量命名（已在本项目 `env.example` 中使用）：

- `LOGTO_ENDPOINT`
- `LOGTO_APP_ID`
- `LOGTO_APP_SECRET`
- `NEXT_PUBLIC_BASE_URL` 或等价的应用 `BASE_URL`
- `LOGTO_COOKIE_SECRET`（如使用 Cookie 加密）
- `NODE_ENV`（控制 Cookie 是否 secure）

## 语言 / 框架集成指引

下面给出不同语言下的典型集成要点和常见坑。**不要复制版本号，使用各语言官方推荐版本。**

### 1. JavaScript / TypeScript（浏览器 / SPA）

**适用场景**：React、Vue、Angular、纯前端 SPA 等。

- 使用 Logto 提供的前端 SDK（如 `@logto/browser`、`@logto/react`）或官方 Quick Start。
- 核心步骤：
  1. 在前端初始化 Logto 客户端，传入：
     - `endpoint`
     - `appId`
     - `redirectUri`
     - `scopes`
     - 可选 `resources`
  2. 在需要登录的地方调用 `signIn()`，SDK 会自动跳转。
  3. 在回调路由解析 Code 并完成 Token 交换。
  4. 使用 `isAuthenticated` / `getAccessToken` 获取登录状态和 Token。
  5. 调用后端 API 时，在请求头添加 `Authorization: Bearer <access_token>`。

**注意事项**：

- SPA 一般不使用 `appSecret`，属于 Public Client。
- 注意回调 URL 必须与 Logto Console 中配置的完全匹配。

### 2. Next.js / Node.js（后端渲染 & API）

**适用场景**：本项目类似的 Next.js Pages Router / Node.js 后端。

- 优先使用 Logto 的 Next.js / Node SDK，如 `@logto/next`、`@logto/node`。
- 典型模式（参照本项目）：
  - 创建一个封装好的 `logtoClient`：
    - 从环境变量读取 `LOGTO_APP_ID`、`LOGTO_APP_SECRET`、`LOGTO_ENDPOINT`、`BASE_URL`、`LOGTO_COOKIE_SECRET`。
    - 配置 `scopes`（如 `profile`、`email`、`phone`、`custom_data`）。
    - 配置 `cookieSecure`：在生产环境使用 `true`。
  - 在 API Route / 中间件中使用 SDK 提供的 `handleSignIn`、`handleSignOut`、`handleAuthRoutes`。
  - 如本项目所示，实现自定义 `sign-in` 路由以支持 `ui_locales` 等额外参数。

**注意事项**：

- Cookie Secret 必须足够随机、长度至少 32 字符。
- `baseUrl` 必须与外部访问地址一致（含端口），否则回调 URL 不正确。

### 3. Python（FastAPI / Django / Flask 等）

**适用场景**：Python Web 后端。

- 可以选用支持 OIDC 的库（如 `authlib`）结合 Logto：
  1. 配置 OIDC Client：
     - `client_id` ← `appId`
     - `client_secret` ← `appSecret`（传统 Web）
     - `server_metadata_url` ← `https://<endpoint>/.well-known/openid-configuration`
  2. 提供登录路由：
     - 构造授权 URL，重定向到 Logto。
  3. 提供回调路由：
     - 使用库提供的方法交换 Token，解析 ID Token。
  4. 在中间件里验证用户是否登录，并将用户信息挂在请求上下文。

**注意事项**：

- 确保服务器的回调 URL 和 Logto Console 配置保持一致。
- 后端应妥善保存 Refresh Token（如果启用）。

### 4. Java（Spring Boot / Jakarta EE）

**适用场景**：使用 Spring Security OIDC 的 Java 服务。

- 配置 Spring Security 的 OIDC Client：
  - `issuer-uri`：Logto 的 OIDC Issuer（与 `endpoint` 相关）
  - `client-id`：`appId`
  - `client-secret`：`appSecret`
  - `scope`：如 `openid profile email`
- 启用 OAuth2 Login：
  - 使用 `oauth2Login()`（前端登录场景）
  - 或使用 Resource Server 模式验证 Bearer Token。

**注意事项**：

- Key / JWK Endpoint 会从 OIDC Discovery 自动获取，只需正确设置 Issuer。
- 确保 `redirect-uri` 模板与 Logto 应用中的回调地址一致。

### 5. Go

**适用场景**：Go Web 服务或 API 网关。

- 使用 Logto Go SDK 或通用 OIDC 库：
  1. 初始化 OIDC Provider（指向 Logto 的 `/.well-known/openid-configuration`）。
  2. 创建 OAuth2 客户端：
     - `ClientID` ← `appId`
     - `ClientSecret` ← `appSecret`
  3. 登录时重定向到授权端点。
  4. 回调路由中用 Code 换 Token，校验 ID Token。
  5. 对受保护 API：
     - 从 `Authorization` 头获取 Bearer Token
     - 使用 OIDC 库验证 Token（签名 / audience / issuer）。

**注意事项**：

- 注意处理 Token 过期；可以使用 Refresh Token 或重新登录。

### 6. .NET（ASP.NET Core）

- 使用 ASP.NET Core Authentication 的 OIDC Handler：
  - 配置 `Authority` 为 Logto Issuer
  - `ClientId`、`ClientSecret` 分别对应 Logto 的 `appId` 和 `appSecret`
  - `ResponseType` 使用 `code`
  - 配置 `CallbackPath` 与 Logto 回调地址匹配。
- 使用 Cookie + OIDC 组合：
  - OIDC 负责与 Logto 交互
  - Cookie 保存本地会话。

**注意事项**：

- appsettings 中的配置应通过 Secret / 环境变量管理。
- 如需调用 API，可使用 `AddJwtBearer` 验证 Access Token。

## 本项目中常见模式（可借鉴）

- **多语言登录界面**：
  - 在前端根据当前语言在登录时附带 `ui_locales` 参数（如 `zh-CN`、`en`），如本项目在 `/api/logto/sign-in` 手动注入 `extraParams`。
- **统一 Auth 路由**：
  - 通过 `handleAuthRoutes` 将 `sign-in-callback`、`user` 等路由集中处理。
- **Subscription / Profile 结合**：
  - 先通过 Logto 获取用户基础信息，再结合本地数据库的订阅信息（本项目中通过 PostgreSQL 实现）。

## 编写 / 修改代码时的注意事项

当用户要求“接入 Logto”或“修改登录逻辑”时，按以下步骤执行：

1. **识别应用类型**（SPA / 传统 Web / API / Mobile）。
2. **定位配置文件 / 环境变量**：
   - 优先使用已有命名（如 `LOGTO_ENDPOINT`、`LOGTO_APP_ID` 等）。
3. **选择对应语言 / 框架的集成策略**（参考上面的语言章节）。
4. **实现或修改以下部分**：
   - 登录入口（Sign-in 路由 / 按钮逻辑）
   - 授权回调处理（解析 Code、换 Token、保存会话）
   - 登出逻辑（清 Cookie + 调用 Logto 端点）
   - 受保护资源的访问控制（中间件 / 拦截器 / Guard）
5. **验证集成**：
   - 使用浏览器或 HTTP 客户端完整走一遍登录 → 回调 → 访问受保护资源 → 登出流程。
   - 检查 ID Token / Access Token 是否来自 Logto 且校验通过。

## 输出格式建议

当向用户输出集成方案或代码示例时：

- **先给清晰的分步说明**（环境变量、配置、路由、保护资源）。
- 再提供 **对应语言的最小可行示例**（只展示关键代码，不要整文件拷贝）。
- 避免硬编码机密信息，使用环境变量占位符（如 `LOGTO_APP_SECRET`）。

