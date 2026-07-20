# CRLF 注入与 HTTP 响应拆分

## 一句话理解
CRLF（`\r\n`，即 `%0d%0a`）是 HTTP 协议里"头部行结束"的分隔符。如果用户输入未经处理进入响应头，攻击者就能人为制造换行，拆分响应、注入头部甚至伪造页面内容。

## 核心原理
- HTTP 响应由"状态行 + 响应头 + 空行 + 响应体"组成，头部之间以 `\r\n` 分隔
- 服务端把用户输入写进响应头（如 `Location`、`Set-Cookie`）时未过滤换行符
- 注入 `%0d%0a` 后，攻击者可以终止当前头、新增任意头，甚至插入空行后伪造整个响应体

## 成立条件
- 输入可控且进入 HTTP 响应头（或日志等以换行为分隔的上下文）
- 服务端/中间件未对 `\r`、`\n` 做过滤或编码

## 常见触发点
- 302 跳转：`/redirect?url=http://a.com`，url 写入 `Location` 头
- Cookie 写入：`Set-Cookie: token=<用户输入>`
- 自定义头回显：`X-Forwarded-For`、`X-Request-Id` 的反射
- 文件名写入：`Content-Disposition: filename=<用户输入>`
- 日志记录：输入未过滤直接写日志（日志伪造/注入）

## 常见利用形式
### 1. HTTP 响应拆分 -> XSS

```text
/redirect?url=%0d%0a%0d%0a<script>alert(1)</script>
```

注入空行后，后续内容被浏览器当作响应体解析，直接执行脚本。

### 2. 注入响应头

```text
%0d%0aSet-Cookie:%20admin=true
```

可种植 Cookie（配合会话固定）、篡改缓存策略、注入安全头以外的任意头。

### 3. 缓存投毒入口
通过注入的头/响应体污染共享缓存（CDN、反向代理），影响后续所有用户——与 [EXP-Request-Smuggling](./EXP-Request-Smuggling.md) 的缓存投毒思路同源。

### 4. 日志伪造
在日志中注入换行，伪造日志条目掩盖攻击痕迹或栽赃他人（蓝队视角也需注意）。

## 绕过手法
| 场景 | Payload |
| --- | --- |
| 基础 | `%0d%0a` |
| 过滤了 `%0d%0a` | `%0a`、`%0d` 单用（部分解析器接受） |
| 双重解码 | `%250d%250a` |
| Unicode 编码差异 | `%e5%98%8a%e5%98%8d`（旧 Java 容器把特定 UTF-8 解码成换行，已少见） |

## 快速判断
1. 找所有会进入响应头的参数（跳转、Cookie、文件名）
2. 注入 `%0d%0aX-Test:%20crlf`，看响应里是否出现 `X-Test: crlf` 头
3. 出现即成立，再评估能打 XSS、投毒还是只能自嗨（Self-XSS 危害需论证）

## 与相邻篇目的关系
- 请求走私本质也是"请求边界解析不一致"，见 [EXP-Request-Smuggling](./EXP-Request-Smuggling.md)
- CRLF 打 XSS 时后续利用与 [EXP-XSS](./EXP-XSS.md) 一致

## 防御要点
- 写入响应头前过滤/编码 `\r`、`\n`
- 框架升级：现代框架默认拒绝头中的换行（老 Tomcat、老 PHP 需自行处理）
- 日志统一 JSON 结构化输出，避免换行注入

## 参考
- [OWASP CRLF Injection](https://owasp.org/www-community/vulnerabilities/CRLF_Injection)
