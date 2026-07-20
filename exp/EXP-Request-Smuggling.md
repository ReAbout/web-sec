# HTTP 请求走私（HTTP Request Smuggling）

## 一句话理解
前端代理和后端服务器对"一个请求在哪里结束"理解不一致，攻击者把一个"暗藏前缀"的请求塞进 TCP 连接，让后端把残留部分拼到**下一个用户的请求**前面，从而越权、投毒、劫持他人会话。

## 核心原理
HTTP/1.1 有两种声明请求体长度的方式：
- `Content-Length`（CL）：字节数
- `Transfer-Encoding: chunked`（TE）：分块编码

链路上只要前端（CDN/反代）和后端对两者的优先级理解不同，边界就会错位：

| 类型 | 前端认 | 后端认 | 结果 |
| --- | --- | --- | --- |
| CL.TE | Content-Length | Transfer-Encoding | 后端把剩余字节当下一请求的开头 |
| TE.CL | Transfer-Encoding | Content-Length | 同上，方向相反 |
| TE.TE | 混淆后的 TE | 另一种 TE | 头混淆（大小写、空格、重复头）绕过一致性 |

与 [EXP-DNS-Rebinding](./EXP-DNS-Rebinding.md) 同属"不一致性"：都是两个组件对同一数据的解析出现分歧。

## 成立条件
- 存在"前置代理/CDN + 后端"的多层架构（单体直连无此问题）
- 前后端使用 HTTP/1.1 keep-alive 连接复用（请求会落在同一条连接上）
- 两端对 CL/TE 优先级处理不一致，或对歧义头未做规范化

## 检测思路
### 1. 时间差法（安全，优先用）
CL.TE 探测：发送后若后端按 chunked 等待下一个块（永远等不到），响应超时：

```http
POST / HTTP/1.1
Host: target.com
Content-Length: 6
Transfer-Encoding: chunked

0

X
```

### 2. 差异响应法
构造使后端把残留拼到下一个请求，观察后续请求是否收到异常响应（404 变 200、多了前缀）。

注意：差异法可能污染真实用户的请求，测试时尽量用时间差法先行确认。

## 常见利用形式
### 1. 绕过前端访问控制
前端禁止访问 `/admin`，但走私的请求不经过前端规则匹配，直接在后端被解析执行。

### 2. 捕获他人请求（会话劫持）
把走私前缀构造成一个不完整的 POST（如评论提交），下一个用户的完整请求被拼进去当作该 POST 的 body——Cookie、Token 全落进攻击者可见的存储里。

### 3. Web 缓存投毒
走私触发异常响应被缓存，正常用户访问同一资源时拿到攻击者构造的内容（可挂 XSS）。

### 4. 绕过前端安全控制打内网
走私请求到达后端时可带上前端才会附加的内部头，或直接访问内部路由。

## HTTP/2 走私
- 前端 HTTP/2、后端 HTTP/1.1 的**降级链路**是现代走私主战场（H2.CL / H2.TE）
- HTTP/2 自身用帧定长，但降级时后端按 CL/TE 解析，差异重新出现
- 头部伪字段（`:method`、`:path`）与降级头的转换也可能引入注入点

## 工具
- Burp Suite 扩展 **HTTP Request Smuggler**：自动探测 CL.TE / TE.CL
- [smuggler](https://github.com/defparam/smuggler)：Python 命令行探测
- Burp Repeater 手工：需关闭 "Update Content-Length"，用 HTTP/1.1 发送

## 快速判断流程
1. 确认架构存在代理层（CDN、Nginx、云 WAF）
2. 时间差法分别探测 CL.TE 与 TE.CL
3. 确认后换差异响应法验证可利用性
4. 评估能映射到哪类利用：绕 ACL / 缓存投毒 / 会话捕获

## 防御要点
- 前后端统一使用 HTTP/2 端到端，避免降级
- 前置代理规范化歧义请求：拒绝同时含 CL 和 TE 的请求
- 后端禁用连接复用（或对复用连接做严格边界校验）可根治但影响性能
- 保持组件更新（历次 CVE 修复了大量解析差异）

## 参考
- [PortSwigger - HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling)
- [请求走私总结@chenjj](https://github.com/chenjj/Awesome-HTTPRequestSmuggling)
