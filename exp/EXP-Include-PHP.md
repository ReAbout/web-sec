# 包含漏洞 PHP

## 一句话理解
文件包含漏洞的核心是应用把用户可控路径传给 `include`、`require` 等函数，导致攻击者能够读取本地文件、包含远程资源或把已写入的恶意内容当成 PHP 代码执行。

## 危险函数
- `include()`
- `include_once()`
- `require()`
- `require_once()`

## 漏洞分类
### 1. LFI
LFI（Local File Inclusion，本地文件包含）是指攻击者控制包含路径，读取或执行本地文件内容。

### 2. RFI
RFI（Remote File Inclusion，远程文件包含）是指包含远程地址上的资源。

RFI 一般需要：
- `allow_url_fopen = On`
- `allow_url_include = On`

现代 PHP 环境中远程包含已相对少见，但 LFI 依然很常见。

## 常见成因
- 直接使用 `$_GET['file']`、`$_REQUEST['page']` 拼接模板路径
- 对文件名只做了弱过滤，例如只替换 `../`
- 业务支持主题、模板、插件动态加载
- 开发误以为“只能包含 `.php`”就安全

## 常见危害
- 读取配置文件、源码、凭证
- 读取日志、Session 文件、环境变量
- 配合日志注入、Session 注入实现代码执行
- 配合 `php://filter` 获取源码
- 配合 `phar://` 触发反序列化链

## 常见利用思路
### 1. 目录穿越
最常见的入口是通过路径穿越包含目标文件，例如：

```text
../../../../etc/passwd
```

### 2. 利用 PHP 包装器
PHP 提供多种 stream wrapper，在包含场景下经常非常关键。

### 3. 利用“可控文件内容”
如果攻击者能把 PHP 代码写入日志、Session、上传文件、临时文件，再通过 LFI 包含该文件，就可能获得代码执行。

## 常见包装器
### 1. `php://input`
可以访问原始请求体，在某些环境下可把 POST 内容作为 PHP 代码包含执行。

前提：
- 需要开启 `allow_url_include=On`
- 对 `allow_url_fopen` 无强依赖

示例：

```text
http://127.0.0.1/index.php?file=php://input
```

POST 数据：

```php
<?php phpinfo(); ?>
```

### 2. `php://filter`
最常用于源码读取，而不是直接执行。

前提：
- 只做读取时，通常只需要 `allow_url_fopen`

示例：

```text
http://127.0.0.1/index.php?file=php://filter/read=convert.base64-encode/resource=xxx.php
```

### 3. `zip://`
可以访问压缩包中的文件。

前提：
- 通常需要 PHP 版本支持
- `#` 常需编码为 `%23`

示例：

```text
http://127.0.0.1/index.php?file=zip://test.zip#shell.php
```

### 4. `phar://`
与 `zip://` 类似，但路径分隔方式不同。

示例：

```text
http://127.0.0.1/index.php?file=phar://test.phar/shell.php
```

> 在很多场景里，`phar://` 的重点不只是“包含文件”，更是“触发反序列化元数据”。

### 5. `data://`
可以直接把数据内容作为包含目标。

前提：
- `allow_url_fopen` 与 `allow_url_include` 均需开启

示例：

```text
http://127.0.0.1/index.php?file=data:text/plain,<?php phpinfo(); ?>
http://127.0.0.1/index.php?file=data:text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+
```

### 6. `file://`
用于访问本地文件系统，一般不受 `allow_url_fopen` 与 `allow_url_include` 影响。

示例：

```text
http://127.0.0.1/index.php?file=file://文件绝对路径
```

## 常见二次利用点
### 1. Session 文件
前提：
- Session 文件路径已知
- Session 内容部分可控

常见路径：
- `/var/lib/php/sess_PHPSESSID`
- `/tmp/sess_PHPSESSID`
- `/tmp/sessions/sess_PHPSESSID`

文件名通常为 `sess_[PHPSESSID]`。

### 2. Web 日志
若能把 PHP 代码写进访问日志、错误日志，再通过 LFI 包含该日志，就可能实现代码执行。

常见日志位置：
- `/var/log/apache2/access.log`
- `/var/log/apache2/error.log`
- Nginx 日志目录

### 3. SSH 登录日志
前提：
- 日志路径已知
- 日志可读

常见位置：
- `/var/log/auth.log`

示例：

```text
$ ssh '<?php phpinfo(); ?>'@remotehost
```

### 4. `/proc/self/environ`
前提：
- PHP 以 CGI 等方式运行，环境变量中保留可控头部
- 文件路径可读

示例：

```http
GET /index.php?file=../../../../proc/self/environ HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 <?phpinfo(); ?>
Connection: close
```

### 5. 上传目录
如果可以先上传可控文件，再通过 LFI 去包含上传文件，也可能实现执行。

## 常见绕过思路
### 1. 路径穿越绕过
- 双写 `../`
- 编码绕过
- 路径截断与标准化差异

### 2. 后缀限制绕过
若代码强制拼接 `.php`，仍需关注：
- 已有 `.php` 文件是否可控
- 日志或 Session 文件是否可被包含
- 包装器是否还能利用

### 3. 黑名单绕过
只过滤 `http://`、`../`、`php://` 通常都不够，攻击者往往能通过编码、包装器和其他文件落点继续利用。

## 实战排查思路
### 1. 先找包含入口
高频参数名：
- `file`
- `page`
- `template`
- `lang`
- `module`
- `inc`

### 2. 再确认包含结果
判断是：
- 只读取报错
- 真正执行
- 可控回显
- 盲包含

### 3. 再找二次利用链
优先确认是否存在：
- 日志可写
- Session 可控
- 上传点
- `phar://` 可达

## 防御要点
### 1. 不要把用户输入直接传给包含函数
模板、语言包、模块名应使用固定映射关系，而不是拼接文件路径。

### 2. 严格白名单
仅允许预定义文件标识，不允许任意路径输入。

### 3. 关闭危险配置
若无业务需要，关闭：
- `allow_url_include`
- `allow_url_fopen`

### 4. 路径规范化与目录隔离
把可包含文件限制在固定目录，规范化后再校验。

### 5. 日志与 Session 不可执行
避免日志、Session、上传目录被当成 PHP 代码执行。

## 速查清单
- 先找 `include`、`require`、模板切换、语言包切换入口
- 先测本地文件读取，再测试包装器
- 重点看 `php://filter`、日志、Session、`/proc/self/environ`
- 若可上传文件，联动检查包含链
- 若存在 `phar://`，进一步检查反序列化利用面

## Reference
- [php 文件包含漏洞](https://chybeta.github.io/2017/10/08/php%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E/)
- [浅谈文件包含漏洞](https://xz.aliyun.com/t/7176)
