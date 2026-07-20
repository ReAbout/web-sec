# 漏洞利用索引（1day 速查表）

## 使用说明
面向渗透与攻防演练的组件漏洞速查索引：先通过指纹识别组件（见 [PEN-Scanner](../penetration/PEN-Scanner.md)），再到本表定位代表漏洞与利用入口。
仅用于授权测试。收录原则：**高频出现 + 有成熟利用链**，新出的 nday 及时补充。

## Java 组件与框架
| 组件 | 代表漏洞 | 识别特征 | 利用入口 |
| --- | --- | --- | --- |
| Shiro | Shiro-550 硬编码密钥反序列化；Shiro-721 Padding Oracle | Cookie 含 `rememberMe` | [ShiroExploit](https://github.com/feihong-cs/ShiroExploit-Deprecated)、[shiro-exploit](https://github.com/Ares-X/shiro-exploit)；原理见 [EXP-Java-Unserialize](./EXP-Java-Unserialize.md) |
| Fastjson | autotype 反序列化（多版本绕过链） | JSON 接口、报错含 `com.alibaba.fastjson` | [FastjsonExploit](https://github.com/c0ny1/FastjsonExploit)、[姿势集合](https://github.com/safe6Sec/Fastjson) |
| Log4j2 | CVE-2021-44228 JNDI 注入（Log4Shell） | 参数回显进日志（UA、Referer、表单字段） | JNDI 利用见 [EXP-Java-Unserialize](./EXP-Java-Unserialize.md) |
| Spring | Spring4Shell（CVE-2022-22965）、SpEL 注入 | Spring Boot、报错含 `org.springframework` | [EXP-SPEL-Injection](./EXP-SPEL-Injection.md)、[SpringBootVulExploit](https://github.com/LandGrey/SpringBootVulExploit) |
| Struts2 | S2-045/046/057/061 OGNL RCE | `.action`/`.do` 后缀、报错含 OGNL | [EXP-OGNL-Injection](./EXP-OGNL-Injection.md)、[Struts2-Scan](https://github.com/HatBoy/Struts2-Scan) |

## 中间件
| 组件 | 代表漏洞 | 识别特征 | 备注 |
| --- | --- | --- | --- |
| Weblogic | T3/IIOP 反序列化（CVE-2015-4852 起系列）、CVE-2020-14882 | 7001 端口、`/console` | 工具：woodpecker、WeblogicScan |
| JBoss | JMXInvokerServlet 反序列化、未授权部署 | 8080、`/jmx-console` | 老牌靶场常客 |
| Tomcat | AJP 文件包含（CVE-2020-1938）、弱口令部署 war、PUT 上传（CVE-2017-12615） | 8009 AJP 端口、`/manager/html` | 配 [EXP-Upload-JSP](./EXP-Upload-JSP.md) |
| Nginx | 解析漏洞（老）、配置错误导致的目录穿越、CRLF | Server 头 | 多为配置问题而非 CVE |
| IIS | PUT 写文件（老 IIS6）、解析漏洞（`;.asp`、`/x.asp/`） | Server 头、短文件扫描 | 结合上传篇 |

## PHP 框架与 CMS
| 组件 | 代表漏洞 | 识别特征 |
| --- | --- | --- |
| ThinkPHP | 5.0.x/5.1.x RCE（路由+invokefunction）、多语言文件包含 | 报错页标志性笑脸/`ThinkPHP Vx.x` |
| Laravel | debug 模式 RCE（CVE-2021-3129 Ignition） | `X-Powered-By`、报错页 Ignition |
| Drupal | Drupalgeddon2（CVE-2018-7600） | `CHANGELOG.txt`、指纹头 |
| WordPress | 插件/主题漏洞为主（Elementor、WooCommerce 系列） | `wp-content` 路径 |
| 织梦 Dedecms | 全版本系列漏洞（会员、文件上传、SQLi） | [dedecmscan](https://github.com/lengjibo/dedecmscan) |
| Discuz | 历史注入/RCE 系列 | `forum.php`、静态化特征 |

## OA 与商用系统
| 系统 | 方向 | 工具 |
| --- | --- | --- |
| 泛微 e-cology / e-office | SQL 注入、文件上传、反序列化 | [OA-EXPTOOL](https://github.com/LittleBear4/OA-EXPTOOL) |
| 致远 OA | 文件上传、未授权、反序列化 | 同上 |
| 蓝凌 OA | 反序列化、任意文件读取 | 同上 |
| 用友 NC | 反序列化、文件上传 | 同上 |
| 通达 OA | 未授权+文件上传组合 RCE | 同上 |

## 其他高频目标
| 组件 | 代表漏洞 | 备注 |
| --- | --- | --- |
| Confluence | CVE-2021-26084、CVE-2022-26134、CVE-2023-22515/22527 | [ConfluenceMemshell](https://github.com/Lotus6/ConfluenceMemshell) |
| Exchange | ProxyLogon（CVE-2021-26855 SSRF+RCE）、ProxyShell | [EBurstGo](https://github.com/X1r0z/EBurstGo) 爆破 |
| GitLab | 任意文件读取、RCE 系列 | CI/CD 场景高价值 |
| Redis | 未授权访问、主从复制 RCE | [EXP-DB-Redis](./EXP-DB-Redis.md) |
| 各类 VPN/网关 | 深信服、奇安信、Fortinet 等历史 RCE | 参考 [redteam_vul](https://github.com/r0eXpeR/redteam_vul) |

## 使用建议
1. 指纹 -> 本表 -> 选漏洞 -> 先验证（dnslog 出网、写文件回读）再深挖
2. 打之前确认版本号，很多 exp 严格依赖版本
3. 无回显场景统一走 dnslog/httplog（见 README dnslog 平台清单）

## 维护规则
- 按组件归类，新组件按表中格式追加
- 出现重大 nday（类似 Log4Shell 级别）时置顶到对应分类首行
- 本地已有专题文档的漏洞，索引只放链接不重复细节
