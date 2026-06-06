# MSSQL SQL注入&漏洞利用

## 一句话理解
MSSQL 注入除了常规数据读取外，往往更值得关注数据库扩展能力、系统存储过程和命令执行组件，因为一旦权限足够，利用链很容易从数据库层直接打到主机层。

## 常见危害
- 登录绕过、数据读取
- 枚举库表列和账号权限
- 报错、布尔、时间型无回显利用
- 调用扩展存储过程执行系统命令
- 配合提权组件进一步拿系统权限

## 基础特点
与 MySQL 相比，MSSQL 的利用重点常在：
- 系统表和系统视图
- 扩展存储过程
- 堆叠查询支持
- `xp_cmdshell`、`sp_configure` 等高危能力

推荐：
- [MSSQL 注入与提权方法整理](https://www.geekby.site/2021/01/mssql%E6%B3%A8%E5%85%A5%E4%B8%8E%E6%8F%90%E6%9D%83%E6%96%B9%E6%B3%95%E6%95%B4%E7%90%86/)

## 常见利用方向
### 1. 数据枚举
重点枚举：
- 当前数据库
- 当前用户
- 数据库版本
- 表、列、权限

### 2. 堆叠查询
MSSQL 在很多场景下更容易遇到堆叠查询利用，例如：
- `;`
- 多语句执行
- 调用存储过程

### 3. 扩展存储过程
一旦权限允许，MSSQL 的扩展存储过程会显著扩大攻击面。

## 命令执行
### `xp_cmdshell`
这是 MSSQL 最经典的高危能力之一，可直接执行操作系统命令。

#### 1. 判断组件是否存在

```sql
select count(*) from master.dbo.sysobjects where xtype='x' and name='xp_cmdshell'
```

#### 2. 开启 `xp_cmdshell`

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

#### 3. 执行命令

```sql
exec master..xp_cmdshell 'whoami'
```

#### 4. 恢复被删除组件

```sql
Exec master.dbo.sp_addextendedproc 'xp_cmdshell','D:\\xplog70.dll'
```

> 实战中是否能成功，强依赖当前数据库权限、SQL Server 配置和宿主机环境。

## 实战排查思路
### 1. 先确认是否可注入
和其他数据库一样，先判断：
- 闭合方式
- 是否有回显
- 是否支持堆叠查询

### 2. 再判断权限边界
MSSQL 的危险性高度依赖权限，重点确认：
- 是否为高权限用户
- 是否可执行存储过程
- 是否可修改配置

### 3. 最后再看主机层利用
优先判断：
- `xp_cmdshell`
- 代理作业
- OLE 自动化
- 其他高危扩展能力

## 防御要点
### 1. 参数化查询
这仍是数据库注入防御的核心。

### 2. 关闭或限制高危组件
尤其是：
- `xp_cmdshell`
- 高危扩展存储过程
- 不必要的外部调用能力

### 3. 最小权限
应用数据库账户不应拥有：
- `sysadmin`
- 修改配置权限
- 执行高危扩展过程权限

### 4. 隐藏详细报错
避免把 MSSQL 错误栈、对象名、过程名暴露给前端。

## 速查清单
- 先判断是否支持堆叠查询
- 先枚举版本、用户、权限
- 重点检查 `xp_cmdshell` 是否存在、是否可启用
- 结合当前账户权限评估能否从库层打到主机层
- 不只盯数据读取，还要看系统过程和提权链

## Reference
- [MSSQL 注入与提权方法整理](https://www.geekby.site/2021/01/mssql%E6%B3%A8%E5%85%A5%E4%B8%8E%E6%8F%90%E6%9D%83%E6%96%B9%E6%B3%95%E6%95%B4%E7%90%86/)
- https://www.freebuf.com/vuls/276814.html
- https://xz.aliyun.com/t/7534
