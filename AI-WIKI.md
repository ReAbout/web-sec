# AI / Wiki 索引

## 目标
这个文件不是替代原笔记，而是给“人类快速定位”和“AI 作为知识库检索”提供统一入口。

适用方式：
- 人工查询：先看主题映射，再跳转到对应专题文档。
- AI 检索：先识别主题、场景、技术栈、认证方式，再按“推荐输出结构”组织答案。

## 文档分层
| 层级 | 目录 | 作用 |
| --- | --- | --- |
| 原理层 | `./vul/` | 解释漏洞边界、成因、上下文 |
| 利用层 | `./exp/` | 记录利用方式、排查点、payload、工具 |
| 流程层 | `./penetration/` | 记录信息收集、权限获取、维持、后渗透 |
| 总控层 | `./README.md` | 人工总目录与高频入口 |

## 主题映射
| 主题 | 别名 / 同义词 | 优先文档 | 常见限定词 |
| --- | --- | --- | --- |
| XSS | 跨站脚本、脚本注入、DOM XSS、CSTI | `./exp/EXP-XSS.md` | `反射型` `存储型` `属性上下文` `CSP` |
| CSRF | 跨站请求伪造、请求伪造、登录 CSRF | `./exp/EXP-CSRF.md` | `SameSite` `Origin` `Token` `Fetch Metadata` |
| SSRF | 服务端请求伪造、内网探测、云 metadata | `./exp/EXP-SSRF.md` | `回显` `盲打` `gopher` `file` |
| SQLi | SQL 注入、数据库注入、盲注、报错注入 | `./exp/EXP-SQLi-MySQL.md` | `MySQL` `Oracle` `MSSQL` `OOB` |
| SSTI | 模板注入、服务端模板注入 | `./exp/EXP-SSTI-ALL.md` | `Jinja2` `Twig` `Freemarker` `Smarty` |
| 命令执行 | RCE、命令注入、代码执行 | `./exp/EXP-CI-PHP.md` `./exp/EXP-CI-Java.md` | `PHP` `Java` `Runtime.exec` |
| XPath 注入 | XML 路径注入 | `./exp/EXP-XPath.md` | `布尔盲注` `认证绕过` |
| XXE | 外部实体注入、XML 外部实体 | `./exp/EXP-XXE.md` | `文件读取` `SSRF` `DTD` |
| 文件上传 | 上传绕过、解析漏洞、WebShell 上传 | `./exp/EXP-Upload.md` | `MIME` `黑白名单` `解析差异` |
| 反序列化 | PHP 反序列化、Java 反序列化、JNDI | `./exp/EXP-PHP-Unserialize.md` `./exp/EXP-Java-Unserialize.md` | `POP` `gadget` `ysoserial` |
| 包含漏洞 | 文件包含、LFI、RFI | `./exp/EXP-Include-PHP.md` | `日志包含` `伪协议` |
| EL / SPEL / OGNL | 表达式注入 | `./exp/EXP-Expression-Injection.md` `./exp/EXP-SPEL-Injection.md` `./exp/EXP-OGNL-Injection.md` | `Java` `Spring` `Struts2` |
| 原型链污染 | Prototype Pollution | `./exp/EXP-nodejs-proto.md` | `__proto__` `merge` `clone` |
| DNS Rebinding | DNS 重绑定 | `./exp/EXP-DNS-Rebinding.md` | `内网打点` `浏览器打内网` |
| 越权 / IDOR | 水平越权、垂直越权、BOLA、未授权访问 | `./exp/EXP-IDOR.md` | `id 遍历` `双账号对比` `资源归属` |
| JWT | Token 伪造、alg=none、算法混淆 | `./exp/EXP-JWT.md` | `HS256` `RS256` `kid` `jku` `弱密钥` |
| Python 反序列化 | pickle、yaml.load、jsonpickle | `./exp/EXP-Python-Unserialize.md` | `__reduce__` `opcode` `PVM` |
| 文件读取 | 目录穿越、任意文件下载、Path Traversal | `./exp/EXP-FileRead.md` | `../` `绝对路径` `伪协议` `%00` |
| NoSQL 注入 | MongoDB 注入、操作符注入 | `./exp/EXP-NoSQL.md` | `$ne` `$gt` `$regex` `$where` |
| CRLF 注入 | HTTP 响应拆分、CRLF | `./exp/EXP-CRLF.md` | `%0d%0a` `Set-Cookie 注入` `响应拆分` |
| 请求走私 | HTTP Request Smuggling、CL.TE | `./exp/EXP-Request-Smuggling.md` | `Content-Length` `chunked` `缓存投毒` |
| CORS / JSONP | 跨域劫持、Origin 反射 | `./exp/EXP-CORS.md` | `ACAO` `credentials` `callback` |
| 信息泄露 | 源码泄露、git 泄露、调试信息 | `./exp/EXP-InfoLeak.md` | `.git` `.env` `swagger` `heapdump` |
| 业务逻辑 | 条件竞争、密码重置、支付篡改 | `./exp/EXP-Logic.md` | `race condition` `越权` `并发` |
| 认证与会话 | Cookie、Session、OAuth、SSO | `./vul/VUL-Auth-Session.md` | `SameSite` `HttpOnly` `会话固定` |
| 密码学误用 | ECB、CBC、Padding Oracle、弱随机 | `./vul/VUL-Crypto.md` | `长度扩展` `比特翻转` `硬编码密钥` |

## 渗透流程映射
| 阶段 | 目标 | 入口文档 |
| --- | --- | --- |
| 网络预置 | 代理、路由、信息收集 | `./penetration/PEN-Openwrt.md` `./penetration/PEN-Tun2socks.md` `./penetration/PEN-Info.md` |
| 网络接入 | 扫描、漏洞验证、1day | `./penetration/PEN-Scanner.md` |
| 权限获取 | 凭证、落地、提权 | `./penetration/PEN-GetHash.md` `./penetration/PEN-GetHash-Linux.md` `./penetration/PEN-Linux-LPE.md` |
| 权限维持 | Shell、WebShell、MSF | `./penetration/PEN-ReShell.md` `./penetration/PEN-Webshell-Question.md` `./penetration/PEN-MSF.md` |
| 隧道代理 | TCP、HTTP、DNS、ICMP | `./penetration/PEN-ssh.md` `./penetration/PEN-Reuse.md` |
| 后渗透 | 内网信息、执行、域 | `./penetration/PEN-WinCmd.md` |
| 域渗透 | 域信息收集、Kerberos 攻击 | `./penetration/PEN-BloodHound.md` `./penetration/PEN-Kerberos.md` |
| 云平台 | AKSK 利用、metadata、对象存储 | `./penetration/PEN-Cloud.md` |
| 反溯源 | 痕迹清理、隐藏 | `./penetration/PEN-LinuxClear.md` `./penetration/PEN-WinClear.md` |

## AI 检索建议
适合先抽取以下维度：
- 漏洞类别：XSS / CSRF / SSRF / SQLi / SSTI / XXE / 反序列化
- 技术栈：`PHP` `Java` `Node.js` `Spring` `Struts2` `Jinja2`
- 认证方式：`Cookie` `Session` `JWT` `Basic Auth`
- 交互限制：`SameSite` `CSP` `CORS` `WAF` `验证码` `二次确认`
- 利用目标：`读文件` `打内网` `提权` `获取凭证` `命令执行`

## 推荐输出结构
AI 回答此仓库问题时，尽量按下面顺序输出：
1. 一句话结论
2. 成立条件
3. 排查点 / 观察点
4. 利用思路或操作步骤
5. 防御要点
6. 对应文档路径

## 高价值问法模板
- `这个接口只有 Cookie 鉴权，没有 Token，也没有 Origin 校验，是否能打 CSRF？`
- `Spring 场景下有哪些常见表达式注入入口，先看哪几个文件？`
- `给我一个从外网打点到提权再到后渗透的阅读路径。`
- `文件上传题，已知后端是 JSP，优先看哪些绕过方式？`
- `看到 __proto__ 可控时，如何快速判断是不是原型链污染？`
- `接口参数里有 id，换 id 能看到别人数据，怎么系统化判断越权？`
- `拿到一个 JWT，按什么顺序测试它的安全性？`
- `进入域内只有普通用户，先做信息收集还是直接打 Kerberoasting？`

## 维护规则
- 新增主题时，同时补“主题映射”里的别名与限定词。
- 新增实战文档时，优先挂到对应阶段或漏洞类别下。
- 如果一个专题既有“原理”又有“利用”，优先保证两边都能跳转到。
