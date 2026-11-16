# 前端安全规范

## 1. 安全边界

前端代码全部运行在用户浏览器里，**用户可以看到、修改、绕过任何前端逻辑**。因此：

- 前端校验、权限隐藏、加密传输的"混淆"，都是体验或成本手段，不是安全边界。
- 真正的鉴权、数据校验、越权防护必须在服务端完成。
- 前端的安全职责是：不主动制造漏洞（XSS、注入、泄露密钥）、不放大后端漏洞造成的后果。

评审时看到"加了个前端校验，所以后端不用校验"，直接阻断。

## 2. XSS

### 2.1 默认安全，显式破例

Vue 的文本插值 `{{ }}` 会转义 HTML，`v-bind` 对属性做转义，这两条路是安全的。风险集中在四处：

| 风险点 | 规则 |
|---|---|
| `v-html` | 默认禁止（ESLint 已设为 error）；确需渲染富文本必须 `DOMPurify.sanitize` |
| `innerHTML` / `document.write` | 禁止 |
| 动态属性拼接 `href` / `src` | 必须校验协议，禁止 `javascript:` |
| 用户输入拼进 `eval` / `new Function` / `setTimeout(字符串)` | 禁止 |

```ts
import DOMPurify from 'dompurify'

export function renderRichText(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a', 'img'],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'title'],
  })
}
```

```vue
<!-- eslint-disable-next-line vue/no-v-html -- 内容已经 DOMPurify 白名单净化 -->
<div v-html="renderRichText(order.remark)" />
```

注意 `eslint-disable` 必须紧邻使用处并写明原因，不能整文件关闭。

### 2.2 URL 协议校验

```ts
export function isSafeUrl(url: string, allowedHosts: string[] = []): boolean {
  try {
    const parsed = new URL(url, window.location.origin)
    if (!['http:', 'https:'].includes(parsed.protocol)) {
      return false
    }
    if (allowedHosts.length === 0) {
      return parsed.origin === window.location.origin
    }
    return allowedHosts.includes(parsed.hostname)
  } catch {
    return false
  }
}
```

用户可控的 `href`、图片 `src`、下载链接都必须过这个校验。

## 3. Token 存储

| 方案 | XSS 风险 | CSRF 风险 | 适用 |
|---|---|---|---|
| `localStorage` + `Authorization` 头 | 有（XSS 可直接读） | 无（不随请求自动携带） | 内部后台，默认方案 |
| `httpOnly` Cookie | 无（JS 读不到） | 有（需 CSRF token / SameSite） | 面向 C 端、高安全要求 |

本项目后台默认 `localStorage`，配套要求：

- token 有效期短（如 2 小时），配合 refresh token 续期，减少泄露窗口。
- 登出、鉴权失败立即清除本地 token 与用户缓存。
- 不在 URL、日志、错误上报中携带 token。
- 严禁把 token 写进源码、`.env`、构建产物中的常量。

选 Cookie 方案时，必须同时满足：`HttpOnly`、`Secure`、`SameSite=Lax`（或 `Strict`）、以及 CSRF token 校验。

## 4. CSRF 与重放

- 使用 `Authorization` 头传 token 时，浏览器不会自动携带，天然免疫传统 CSRF。
- 若使用 Cookie 鉴权：写操作必须带 CSRF token，服务端校验；`SameSite` 不能省。
- 支付、短信验证码、下单等关键接口必须防重放：前端每次生成幂等键或随机 nonce，随请求发送，服务端去重。
- 前端按钮防重复提交只解决误触，不解决重放，不能替代服务端幂等。

## 5. 敏感信息

- **所有 `VITE_` 变量都会打进产物**，任何人打开 DevTools 即可看到。API 密钥、第三方 secret、内网数据库地址一律不写入；需要调第三方时走后端代理。
- `.env.local`、`*.pem`、签名证书不入库，`.gitignore` 已覆盖常见模式。
- 源码中禁止硬编码测试账号密码；演示数据用明显虚构的信息。
- 生产构建移除 `console` 输出（ESLint 只拦新代码，构建期再用 `esbuild.drop` 兜底）：

```ts
// vite.config.ts
export default defineConfig({
  esbuild: {
    drop: ['console', 'debugger'],
  },
})
```

- Sourcemap 生产默认关闭；需要定位线上问题时上传到内网制品库，不放在可公开访问的静态路径。
- 错误上报过滤敏感字段：手机号、身份证、token、卡号，统一在 `reportError` 中脱敏。

```ts
const SENSITIVE_KEYS = ['token', 'password', 'phone', 'idCard', 'authorization']

export function maskSensitive(payload: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(payload).map(([key, value]) => [
      key,
      SENSITIVE_KEYS.some((k) => key.toLowerCase().includes(k.toLowerCase())) ? '***' : value,
    ]),
  )
}
```

## 6. 权限

- 路由守卫与 `v-permission` 只是**展示控制**：让用户看不到无权入口，减少误操作与困惑。
- 真正的越权防护在服务端：每个接口都要校验"当前用户能否操作这条数据"，不能只校验"是否登录"。
- 权限点命名前后端一致（`order:list`、`order:cancel`），避免前端判断通过、后端拒绝的割裂。
- 禁止把权限判断结果缓存成"永久有效"，切换账号或权限变更后必须重新拉取。

```ts
// src/directives/permission.ts
import type { Directive } from 'vue'
import { useUserStore } from '@/stores/user'

export const permission: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    const userStore = useUserStore()
    if (!userStore.hasPermission(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  },
}
```

## 7. 依赖安全

- 提交 `pnpm-lock.yaml`，CI 用 `--frozen-lockfile`，防止构建时偷偷装上新版本。
- 定期 `pnpm audit --prod`，高危漏洞必须升级或替换；无法升级的要在工单中记录缓解措施。
- 新增依赖前确认来源：官方仓库、下载量正常、近期有维护、无异常 `postinstall` 脚本。
- 警惕 `postinstall` 执行任意脚本（供应链攻击的常见载体），必要时用 `pnpm approve-builds` 显式放行。
- 不引入来源不明的 CDN 脚本；必须引入时加 `integrity` 与 `crossorigin`。

```html
<script
  src="https://cdn.example.com/lib.min.js"
  integrity="sha384-xxxx"
  crossorigin="anonymous"
  defer
></script>
```

## 8. 第三方脚本

- 统计、客服、地图等第三方脚本使用 `defer` / `async`，不阻塞首屏。
- 限制脚本来源白名单，由 CSP 或网关控制；不在页面中动态拼 `src`。
- 第三方脚本运行在同源上下文，能读到 `localStorage`：涉及支付、身份的场景要评估风险，必要时隔离到 iframe。
- 建议配置 CSP（由服务端下发响应头）：

```text
Content-Security-Policy: default-src 'self'; img-src 'self' https: data:; script-src 'self' https://cdn.example.com; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.example.com; frame-ancestors 'none'
```

`style-src` 的 `'unsafe-inline'` 通常因组件库动态样式而不得不放开，但要尽量收窄 `script-src`。

## 9. 上传与下载

- 文件类型与大小校验在前端只是体验优化，服务端必须重新校验（MIME、后缀、魔数、大小）。
- 文件名不可信：展示时转义，拼接路径时使用服务端返回的存储 key，不用原始文件名。
- 预览本地图片用 `URL.createObjectURL`，用完必须 `revokeObjectURL`，否则内存泄漏：

```ts
const url = URL.createObjectURL(file)
previewSrc.value = url
onUnmounted(() => URL.revokeObjectURL(url))
```

- 下载接口若返回 `Content-Disposition: attachment`，前端不要再把内容插入 DOM。
- 禁止把用户上传的文件直接当作 HTML/SVG 渲染（SVG 可内嵌脚本）。

## 10. 重定向与跳转

登录后按 `redirect` 参数回跳时，必须校验为同站相对路径，否则会成为开放重定向漏洞（钓鱼链接 `?redirect=https://evil.com`）：

```ts
export function safeRedirect(target: unknown, fallback = '/'): string {
  if (typeof target !== 'string' || !target) {
    return fallback
  }
  // 只允许以单个 / 开头的站内路径
  if (!target.startsWith('/') || target.startsWith('//')) {
    return fallback
  }
  return target
}
```

- `window.open` 与 `<a target="_blank">` 必须带 `rel="noopener noreferrer"`，否则新页面可通过 `window.opener` 操作原页面。
- 外部跳转（支付、帮助文档）白名单校验域名，不用用户输入直接拼 URL。

## 11. 反模式清单

1. `v-html` 渲染用户评论且未净化。
2. 把 API 密钥写进 `VITE_` 变量。
3. 用 `eval` / `new Function` 解析后端返回的"脚本配置"。
4. 用户输入直接拼进 `RegExp`（ReDoS 风险），需要转义特殊字符。
5. `target="_blank"` 不带 `rel`。
6. `localStorage` 存完整用户信息（含身份证、手机号）。
7. 前端用"隐藏按钮"当作权限控制，接口无鉴权。
8. 直接把 `location.href = query.redirect`。
9. 上传后把用户文件名拼进 `innerHTML`。
10. 生产中保留 `console.log` 打印 token 或完整响应体。
