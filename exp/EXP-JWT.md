# JWT 攻击

## 一句话理解
JWT（JSON Web Token）的安全性完全依赖签名验证。一旦服务端在"算法选择、密钥强度、声明校验"任何一环放松，攻击者就能伪造任意身份的 Token。

## JWT 结构速览
三段式，`.` 分隔，均为 Base64Url 编码：

```text
eyJhbGciOiJIUzI1NiJ9   .   eyJ1c2VyIjoiYWRtaW4ifQ   .   <signature>
       header                    payload                   signature
```

- `header`：`alg`（签名算法）、`typ`、可选 `kid`/`jku`/`x5u`
- `payload`：业务声明，如 `sub`、`role`、`exp`、`iat`、`iss`、`aud`
- `signature`：对前两段的签名，保证完整性

## 核心原理
- 签名只保证"没被改过"，**不保证机密性**，payload 任何人可解码
- 攻击的核心思路只有一个：让服务端接受攻击者构造的 Token
- 常见突破口集中在四点：算法头可控、密钥可猜、头部参数被信任、声明不校验

## 常见攻击面
### 1. `alg=none`
将 header 改为 `{"alg":"none"}`，签名字段留空（注意保留末尾的 `.`）。部分库在校验时跳过签名。

### 2. 算法混淆（RS256 -> HS256）
- 服务端本用 RSA 非对称签名（RS256），公钥通常可获取（JWKS 接口、证书、前端 JS）
- 攻击者把 `alg` 改成 HS256，用**公钥作为 HMAC 密钥**签名
- 如果服务端校验逻辑按 header 里的 alg 选验证方式，就会拿公钥去验 HMAC，伪造成功

### 3. 弱密钥爆破
HS256 的密钥如果是弱口令，可离线爆破：

```bash
hashcat -m 16500 jwt.txt dict.txt
```

常见弱密钥：`secret`、`key`、`123456`、项目名、配置文件里的默认值。

### 4. `kid` 注入
`kid` 用于指定密钥 ID，若服务端直接用其拼接文件路径或 SQL：
- 路径穿越：`"kid":"../../dev/null"` 配合已知内容文件做空密钥签名
- SQL 注入：`"kid":"x' UNION SELECT 'secret'--"`

### 5. `jku` / `x5u` 头注入
header 中的 `jku` 指向 JWKS 地址。若服务端不校验域名白名单，攻击者指向自己服务器上的公钥集，用对应私钥签名即可。常配合 SSRF 或重定向绕过域名校验。

### 6. 声明不校验
- 不校验 `exp`：过期 Token 永久有效
- 不校验 `aud`/`iss`：A 系统的 Token 拿到 B 系统用
- 校验顺序错误：先信任 payload 中的 `role` 再做其他处理（逻辑缺陷）

## 快速判断流程
1. 解码 Token，看 `alg`、payload 里有哪些身份字段
2. 改 `alg=none`，看是否被接受
3. 确认算法是 HS 还是 RS；RS 则找公钥，试算法混淆
4. HS256 直接上字典爆破
5. 观察 `kid`/`jku`/`x5u` 是否存在且可控
6. 删掉/篡改签名、过期时间，观察服务端报错差异（泄露校验逻辑）

## 工具
- [jwt_tool](https://github.com/ticarpi/jwt_tool)：一站式测试（alg=none、混淆、爆破、kid 注入）
- [jwt.io](https://jwt.io/)：在线解码调试
- hashcat `-m 16500`、jwt-cracker：离线爆破

## 防御要点
- 服务端白名单固定算法，忽略 header 中的 `alg` 切换请求
- HS256 密钥保持高强度随机，并支持轮换
- 不信任 `kid`/`jku`/`x5u`，密钥来源写死或严格白名单
- 完整校验 `exp`/`nbf`/`iss`/`aud`
- payload 不放敏感信息，必要时使用 JWE 加密

## 参考
- [PortSwigger JWT attacks](https://portswigger.net/web-security/jwt)
- [RFC 7519 - JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
