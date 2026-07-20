# 信息泄露

## 一句话理解
信息泄露本身很少"致命"，但它是几乎所有攻击链的第一步：源码、配置、密钥、接口文档任何一样泄露，都可能把黑盒测试变成白盒审计，把无入口变成有入口。

## 核心特点
- 单看评级低，组合起来杀伤力大：`.git` -> 源码 -> 审计出 RCE -> 配置文件 -> 数据库 -> 内网
- 自动化程度高，适合批量收割（SRC 常见）
- 防御方最容易忽视：上线忘删、备份随手放、debug 忘关

## 常见类型
### 1. 源码泄露
| 类型 | 路径/特征 | 利用 |
| --- | --- | --- |
| Git | `/.git/HEAD`、`/.git/config` | [GitHack](https://github.com/lijiejie/GitHack)、[GitHacker](https://github.com/WangYihang/GitHacker) 还原完整仓库 |
| SVN | `/.svn/entries`、`/.svn/wc.db` | [dvcs-ripper](https://github.com/kost/dvcs-ripper) |
| .DS_Store | `/.DS_Store` | [ds_store_exp](https://github.com/lijiejie/ds_store_exp) 还原目录结构 |
| IDE 配置 | `/.idea/`、`.vscode/`、`*.sublime-project` | 泄露路径、部署信息 |
| 备份文件 | `index.php.bak`、`index.php~`、`index.php.swp`、`www.zip`、`backup.tar.gz` | 直接下载源码 |
| 编辑器临时文件 | Vim 异常退出残留 `.swp/.swo` | 还原文件内容 |

### 2. 配置泄露
- `.env`：数据库口令、AKSK、SMTP、JWT 密钥
- Java：`/WEB-INF/web.xml`、`application.yml`、`bootstrap.properties`
- `.htaccess`、`web.config`、`config.php.swp`
- Spring Boot Actuator：`/actuator/env`、`/actuator/heapdump`（堆 dump 里能翻出口令）

### 3. 接口与文档泄露
- Swagger / OpenAPI：`/swagger-ui.html`、`/v2/api-docs`、`/v3/api-docs`
- GraphQL 内省：`__schema` 查询拿到全部类型定义（关掉 introspection 也难挡字段爆破）
- 接口文档站：Yapi、Apidoc 未授权访问

### 4. 调试信息泄露
- 报错堆栈：泄露绝对路径、框架版本、SQL 语句
- `phpinfo()`：环境变量、模块、路径全泄露
- Flask/Werkzeug debug：`/console` 可交互执行（结合 PIN 计算可 RCE）
- Django `DEBUG=True`：配置、SQL、代码片段全展示

### 5. 前端泄露
- JS 里的硬编码密钥、内网地址、接口路径
- Sourcemap（`.js.map`）：还原前端源码
- HTML 注释里的测试账号、TODO

### 6. 云密钥泄露
- GitHub/Gitee 搜索 `AKID`、`AccessKeyId`、`SecretAccessKey`、`LTAI`（阿里云 AK 前缀）
- APK/小程序反编译、JS bundle 里翻 AKSK -> 接 [PEN-Cloud](../penetration/PEN-Cloud.md)

## 快速判断
1. 敏感路径 fuzz：`/.git/HEAD` 返回 `ref:`、`/.svn/entries` 有内容、`.DS_Store` 是二进制小文件
2. 状态码语义：200/403/404 分别代表"可读/存在但禁读/不存在"，403 也值得换姿势再试
3. 报错触发：传非法参数、超长输入、类型错误，看是否回显堆栈
4. 指纹联动：识别出 Spring 就测 actuator，识别出 PHP 就测 phpinfo 和 `.swp`

## 工具
- 目录扫描：dirsearch、ffuf、dirmap（配合 [PEN-Scanner](../penetration/PEN-Scanner.md)）
- Git 利用：GitHack、GitHacker、git-dumper
- 备份扫描：常见后缀字典 `bak/backup/swp/swo/old/zip/tar.gz/sql`

## 利用链示例（CTF 常见）

```text
/.git 泄露 -> 还原源码 -> 发现 config.php 数据库口令
         -> 审计出反序列化入口 -> 写入 webshell -> getshell
```

## 防御要点
- Web 根目录禁放 `.git/.svn`，CI/CD 构建产物与版本库分离
- 线上关闭 debug、actuator 最小化暴露、错误页统一
- 备份文件移出 Web 目录，密钥进 KMS/环境变量而非仓库
- 网关层拦截敏感路径（`.git`、`.env`、`actuator`）

## 参考
- [OWASP - Information Leakage](https://owasp.org/www-community/vulnerabilities/Information_Leakage)
