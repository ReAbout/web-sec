# 前端安全-CSRF

## 一句话理解
CSRF（Cross-site Request Forgery，跨站请求伪造）是攻击者诱导受害者浏览器在已登录状态下，向目标站点发送“用户本人并未主动发起”的请求。

## 核心原理
浏览器会自动携带和目标站点相关的认证信息，例如 Cookie、部分浏览器凭据、已建立的会话状态。  
如果服务端只校验“请求是否带着有效登录态”，而不校验“请求是否确实由本站页面发起”，就可能出现 CSRF。

## 与 XSS 的区别
- XSS 利用的是用户对站点的信任
- CSRF 利用的是站点对用户浏览器的信任

XSS 一旦成立，往往还能反过来窃取 CSRF Token 或直接调用站内接口，因此很多场景里 XSS 的优先级高于单纯的 CSRF。

## 成立条件
CSRF 通常需要满足以下条件：
- 受害者已经登录目标站点
- 浏览器会自动附带有效认证信息
- 目标操作仅依赖 Cookie 或会话状态鉴权
- 攻击者可以构造一个受害者浏览器能发起的请求
- 服务端没有额外校验请求来源或一次性凭证

## 常见攻击面
- 修改个人资料、绑定邮箱、修改手机号
- 修改密码、重置二次验证、添加信任设备
- 发帖、发消息、点赞、关注、加好友
- 转账、下单、收货地址修改
- 管理后台中的新增管理员、授权、导入导出、Webhook 配置

## 常见利用形式
### 1. GET 请求
如果敏感操作被设计成 GET，利用门槛最低：

```html
<img src="https://target.com/user/delete?id=1">
```

或：

```html
<a href="https://target.com/profile/update?role=admin">click</a>
```

### 2. POST 表单自动提交

```html
<form action="https://target.com/transfer" method="POST">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="1000">
</form>
<script>
document.forms[0].submit();
</script>
```

### 3. 特定 Content-Type 场景
很多站点以为“只允许 POST + JSON 就安全”，但如果接口也接受表单、文本、宽松解析或参数覆盖，就仍可能被构造。

### 4. 登录 CSRF
有些业务不是“冒充受害者执行操作”，而是“把受害者悄悄登录进攻击者控制的账号”，从而影响后续操作和数据流向。

## 为什么有些接口不容易被 CSRF
以下机制会显著提高攻击难度：
- 请求必须带不可预测的 CSRF Token
- Token 绑定用户会话且一次一用
- 服务端严格校验 `Origin` / `Referer`
- Cookie 使用 `SameSite`，浏览器不再在跨站请求中自动附带
- 敏感操作要求再次输入密码、短信验证码、二次确认

## 防御思路
防御的本质：要求请求同时携带“攻击者无法伪造或获取的额外凭证”。

### 1. CSRF Token
最常见、也最有效的方案。

关键要求：
- Token 不可预测
- 与用户会话绑定
- 最好与操作场景绑定
- 不要只校验“字段存在”
- 不要多个用户共享同一个 Token

### 2. SameSite Cookie
通过浏览器机制限制跨站请求自动带 Cookie。

常见选项：
- `SameSite=Strict`
- `SameSite=Lax`
- `SameSite=None; Secure`

> 这是一层很重要的浏览器侧防线，但不应替代 Token 校验。

### 3. 校验 `Origin` / `Referer`
适合作为辅助校验。

优点：
- 实现相对简单
- 对部分无 Token 场景有补充价值

缺点：
- 某些环境下请求头可能缺失
- 不适合作为唯一防线

### 4. 双重 Cookie 校验
思路是：服务端在 Cookie 中放一个值，前端再把同样的值放到请求参数或自定义头中，服务端比对二者是否一致。

适用点：
- 前后端分离场景
- 与 Token 机制组合使用

### 5. 自定义请求头
浏览器跨站表单无法随意添加自定义头，因此接口要求：
- `X-CSRF-Token`
- `X-Requested-With`
- 其他受控头字段

也能提高攻击门槛，但仍建议与 Token 联合使用。

### 6. 敏感操作二次确认
例如：
- 重新输入密码
- 短信验证码
- TOTP 校验
- 确认弹窗 + 服务端状态校验

## 常见绕过思路
### 1. Token 校验缺陷
常见问题包括：
- 删除 Token 也能通过
- Token 只校验长度或格式
- Token 在不同用户之间可以共用
- Token 不绑定会话
- Token 长期不变且可预测
- 只校验 Cookie 中的 Token，不校验请求参数中的 Token

排查点：
- 删除令牌：删除参数或 Cookie 中的 Token，观察是否仍能通过
- 令牌共享：创建两个账户，互换 Token 看是否可复用
- 篡改令牌值：看服务端是否仅做弱校验
- 解码令牌：判断是否只是简单编码，如 Base64、MD5 包裹
- 修改请求方法：把 `POST` 改成 `GET` 看服务端是否同样接受

### 2. Token 泄露
虽然 CSRF 的经典前提是“攻击者无法直接读取响应”，但 Token 依然可能通过以下方式泄露：
- XSS
- 开放重定向
- Web 缓存欺骗
- 日志泄露
- Referer 外带
- Clickjacking 配合交互

### 3. `Referer` / `Origin` 校验缺陷
典型错误：
- 只做“包含目标域名”的字符串判断
- 只判断前缀，不做完整主机名校验
- 正则写得过于宽松

例如：
- `http://baidu.com.reabout.com`
- `http://attack.com?http://baidu.com`

### 4. CORS 误配置联动
严格来说 CORS 不是 CSRF 防御机制，但如果：
- 允许任意源跨域
- 允许携带凭据
- 响应里暴露敏感数据

那么问题会升级为“攻击者不仅能发请求，还能读响应”。

### 5. GET 接口做敏感操作
一旦敏感操作被设计成 GET，请求可被非常低成本地嵌入：
- `img`
- `script`
- `iframe`
- `link`

## 实战排查思路
### 1. 优先找高价值状态变更接口
关注：
- 个人信息修改
- 账户安全设置
- 金融与支付
- 后台权限与配置变更

### 2. 看请求是否只依赖 Cookie
如果无 Token、无来源校验、无二次确认，优先级通常较高。

### 3. 看浏览器侧限制是否存在
重点关注：
- 是否使用 `SameSite`
- 是否必须自定义头
- 是否限制请求类型

### 4. 看是否可由低交互触发
自动提交表单、页面加载即触发的请求，危害通常更高。

## 防御清单
- 敏感操作必须使用 CSRF Token
- Token 必须绑定会话，且不可预测
- 优先校验 `Origin`，缺失时再谨慎校验 `Referer`
- 认证 Cookie 配置 `SameSite`
- 敏感动作避免使用 GET
- 高风险操作增加二次确认
- 不要让前端缓存、日志、跳转把 Token 带出去

## Reference
- [前端安全系列（二）：如何防止 CSRF 攻击？](https://tech.meituan.com/2018/10/11/fe-security-csrf.html)
- [跨站请求伪造（CSRF）挖掘技巧及实战案例全汇总](https://cloud.tencent.com/developer/article/1516424)
