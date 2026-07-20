# 认证与会话机制（原理篇）

## 定位
本篇是原理层文档：讲清楚 Web 应用"怎么记住你是谁"，以及这套机制的边界在哪里。利用手法不展开，分别指向 exp/ 对应篇目。

## 认证 vs 授权
理解大量漏洞的总钥匙：
- **认证（Authentication）**：你是谁？——登录、Token、证书
- **授权（Authorization）**：你能做什么？——角色、权限、资源归属

绝大多数越权漏洞，都是因为系统把两件事混为一谈：验了"登录了没有"，没验"这资源是不是你的"。利用见 [EXP-IDOR](../exp/EXP-IDOR.md)。

## Cookie + Session 模式
### 工作流程
1. 登录成功，服务端创建 Session（存内存/Redis），下发 `Set-Cookie: SESSIONID=xxx`
2. 浏览器之后每次请求自动携带 Cookie
3. 服务端按 SESSIONID 查 Session，恢复身份

### Cookie 关键属性
| 属性 | 作用 | 安全意义 |
| --- | --- | --- |
| `HttpOnly` | JS 不可读 | 缓解 XSS 盗 Cookie（不能缓解 CSRF） |
| `Secure` | 仅 HTTPS 传输 | 防明文嗅探 |
| `SameSite` | 跨站携带策略（Strict/Lax/None） | 现代 CSRF 防线，见 [EXP-CSRF](../exp/EXP-CSRF.md) |
| `Domain` / `Path` | 作用范围 | 设置过宽会被子域滥用，过窄会丢失登录态 |
| `Expires/Max-Age` | 有效期 | 长有效期 + 无服务端失效 = 登出是假的 |

### 经典攻击面
- **会话固定（Session Fixation）**：登录前后 SESSIONID 不变，攻击者先塞一个 ID 给受害者
- **会话预测**：ID 生成弱随机（时间戳、自增），见 [VUL-Crypto](./VUL-Crypto.md)
- **登出不失效**：服务端只清前端 Cookie，Session 仍可复用
- **子域会话共享**：`Domain=.target.com` 时子域 XSS 可读写主站 Cookie

## Token / JWT 模式
### 与 Session 的本质区别
- Session：状态在服务端，可撤销、可控制，但每次查存储
- JWT：状态在客户端（自包含签名），服务端无状态、不可单点撤销，靠短有效期 + 刷新机制兜底

### 边界认知
- JWT 签名只保证完整性，**payload 不保密**
- "退出登录"对 JWT 只是前端删除，Token 到期前仍有效（除非维护黑名单，那就退化回有状态了）
- 利用手法（伪造、混淆、爆破）见 [EXP-JWT](../exp/EXP-JWT.md)

## OAuth 2.0 / OIDC
常用于第三方登录（"微信登录""Google 登录"），流程：跳转授权 -> 回调带 code -> 后端用 code 换 token。

常见误用（漏洞高发区）：
- **缺 `state` 参数**：回调可被 CSRF，攻击者把自己的账号绑给受害者
- **`redirect_uri` 校验不严**：开放重定向 + 授权码/Token 泄露
- **隐式模式（implicit）**：Token 直接出现在 URL，易泄露（历史原因仍在用）
- **绑定逻辑缺陷**：第三方账号与本地账号绑定环节可越权

## SSO / 单点登录
- CAS、SAML、OIDC 都是"一处认证，多处信任"
- 攻击面：票据伪造（SAML 签名绕过）、回调投毒、跨系统权限提升
- 一处失守 -> 全线失守，域渗透里的 SSO 与 Kerberos 同理（见 [PEN-Kerberos](../penetration/PEN-Kerberos.md)）

## 凭证存储（防御侧必须知道）
- 口令绝不明文/可逆加密存储，用 bcrypt / scrypt / Argon2 + salt
- 哈希速度是"缺陷"也是"特性"：快哈希（MD5/SHA1）利于爆破，慢哈希提高成本
- 泄露后的连锁：口令复用 -> 撞库 -> 多平台失守

## 攻击面速查映射
| 机制 | 典型攻击 | 利用篇 |
| --- | --- | --- |
| Cookie 自动携带 | CSRF | [EXP-CSRF](../exp/EXP-CSRF.md) |
| Cookie 可读 | XSS 窃取 | [EXP-XSS](../exp/EXP-XSS.md) |
| Session 可预测/固定 | 会话劫持 | 本篇 + [VUL-Crypto](./VUL-Crypto.md) |
| JWT 签名缺陷 | Token 伪造 | [EXP-JWT](../exp/EXP-JWT.md) |
| 只认证不授权 | 越权/IDOR | [EXP-IDOR](../exp/EXP-IDOR.md) |
| OAuth 回调 | 账号劫持 | 本篇 OAuth 节 |

## 参考
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [RFC 6749 - OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
