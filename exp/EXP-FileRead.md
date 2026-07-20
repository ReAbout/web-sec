# 任意文件读取与目录穿越

## 一句话理解
应用按用户给的路径去磁盘读文件，却没有把路径限制在预期目录内，攻击者用 `../` 跳出沙箱目录，读走配置、源码、密钥甚至 flag。

## 核心原理
- 用户输入（文件名/路径）拼进文件操作函数（`open`、`readFile`、`FileInputStream`）
- 服务端未做规范化校验，相对路径符号 `../`（Windows 下 `..\`）向上回溯目录
- 与"文件包含"的区别：读取只拿内容，包含会把内容**当代码执行**（见 [EXP-Include-PHP](./EXP-Include-PHP.md)）

## 成立条件
- 存在读文件/下载/预览功能，路径或文件名用户可控
- 路径校验缺失、黑名单可绕过，或校验发生在规范化之前

## 常见触发点与参数
- 文件下载：`/download?file=report.pdf`
- 图片/附件预览：`/preview?path=...`、`/img?src=...`
- 模板/语言包加载：`?template=`、`?lang=`
- 常见参数名：`file`、`filename`、`path`、`filepath`、`download`、`doc`、`src`、`name`

## 常见利用形式
### 1. 相对路径穿越

```text
../../../../etc/passwd
....//....//etc/passwd        （过滤一次 ../ 时双写绕过）
..%2f..%2fetc/passwd          （URL 编码）
..%252f..%252f                （双重编码，前端解码一次、服务端再解码一次）
```

### 2. 绝对路径
直接给绝对路径，很多校验只盯 `../`：

```text
/etc/passwd
C:\Windows\win.ini
```

### 3. 截断与后缀绕过
- `%00` 截断（PHP < 5.3.4 等老环境）：`../../etc/passwd%00.jpg`
- 长度截断：超长 `../` 冲刷掉服务端拼接的固定后缀（老 PHP）
- 伪协议（PHP）：`php://filter/convert.base64-encode/resource=index.php`，把源码 Base64 读出来

### 4. Windows 差异
- 分隔符 `\` 与 `/` 都可能被接受
- 盘符路径 `C:/`、`\\?\C:\`
- 保留设备名（`aux`、`con`）可用于 DoS 探测

## 读什么
| 目标 | 价值 |
| --- | --- |
| `/etc/passwd`、`C:\Windows\win.ini` | 探测漏洞是否成立的试金石 |
| `/proc/self/environ`、`/proc/self/cmdline` | 环境变量、启动参数（常藏密钥） |
| `/proc/self/fd/*` | 已打开文件句柄，可读日志/配置 |
| `web.xml`、`.env`、`application.yml`、`config.php` | 数据库口令、AKSK、JWT 密钥 |
| 站点源码 | 拿到源码转代码审计，挖 RCE |
| `/var/log/`、Tomcat 日志 | 找 session、后台路径、注入痕迹 |
| `~/.ssh/id_rsa`、`~/.bash_history` | 直接登录、摸清运维习惯 |

## 进阶思路
- 读源码 -> 审计出反序列化/SQLi -> 组合 RCE，是 CTF 与实战的标准链路
- 读取数据库文件（SQLite）、`/proc/self/maps` 辅助进一步利用
- 盲读场景：无回显时结合布尔差异（存在/不存在响应不同）或 OOB 外带

## 快速判断
1. 找读文件类参数，先丢 `../../../../etc/passwd` 看回显
2. 被拦就换编码、双写、绝对路径逐一试
3. 回显过滤了敏感关键词时，用 Base64 伪协议读出再本地解码

## 防御要点
- 白名单校验文件名（只允许预期集合），不校验"路径"而校验"文件 ID"
- 拼接后用 `realpath` / 路径规范化函数处理，确认结果仍以预期目录开头
- 拒绝绝对路径与 `..` 序列，校验放在 URL 完全解码之后
- 单独隔离存储目录，Web 用户最小权限，禁读系统目录

## 参考
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [PortSwigger File path traversal](https://portswigger.net/web-security/file-path-traversal)
