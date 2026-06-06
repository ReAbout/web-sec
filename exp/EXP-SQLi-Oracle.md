# SQL 注入 Oracle

## 一句话理解
Oracle 注入与 MySQL 注入思路相通，但在语法、系统表、伪表、字符串拼接、时间盲注和报错利用方式上有明显差异，不能直接套用 MySQL payload。

## 常见差异
- Oracle 常用伪表是 `dual`
- 字符串拼接常用 `||`
- 没有 `limit`，常用 `rownum`
- 系统信息主要从 `user_tables`、`all_tables`、`dba_tables` 等视图获取
- 时间盲注、报错注入、网络外带方式与 MySQL 不同

## 常见危害
- 登录绕过
- 查询当前用户、版本、库对象
- 枚举表、列、数据
- 通过报错或时间差做无回显利用
- 在高权限和特定组件场景下做网络请求或更深层利用

## 基础探测
### 1. 判断是否进入 SQL 结构
常见探针：
- `'`
- `and 1=1`
- `and 1=2`

### 2. 用 `dual` 做表达式测试

```sql
SELECT 1 FROM dual
SELECT 'test' FROM dual
```

如果能注入子查询、条件表达式或 Union，后续利用空间会大很多。

## 常见枚举思路
### 1. 当前用户

```sql
select user from dual
```

### 2. 数据库版本

```sql
select banner from v$version
```

### 3. 当前用户可见表

```sql
select table_name from user_tables
select table_name from all_tables
```

### 4. 查询列名

```sql
select column_name from user_tab_columns where table_name='USERS'
```

> Oracle 中对象名默认常为大写，枚举时要特别注意大小写。

## Union 注入
### 1. 判断列数
可通过 `ORDER BY` 或逐步 `UNION SELECT` 判断字段数。

### 2. 注意类型匹配
Oracle 对 Union 两边的字段类型要求更严格，若原查询列类型不一致，往往需要显式构造：
- 字符串列
- 数值列
- `null` 占位

### 3. 常见写法

```sql
union select null,null from dual
union select 'a',null from dual
```

## 报错型注入
Oracle 也存在基于类型转换、XML 处理、函数异常等思路的报错利用，但具体利用函数受版本、权限和过滤器影响较大。

实战要点：
- 关注类型转换错误
- 关注 XML 相关函数报错
- 关注子查询返回行数异常
- 关注 `utl_inaddr`、`utl_http`、`extractvalue` 等是否可用

## Bool 型盲注
如果页面只存在真假差异，可用条件判断逐字符枚举。

例如判断首字符：

```sql
and ascii(substr((select user from dual),1,1))=83
```

## 时间型盲注
Oracle 常见思路是借助耗时函数、锁等待或网络请求形成时间差，和 MySQL 的 `sleep()` 思路不同。

实战重点：
- 关注 `dbms_lock.sleep`
- 关注网络函数与 DNS / HTTP 外带
- 关注是否可构造条件触发耗时逻辑

> 是否能直接调用相关包，强依赖数据库权限。

## 常见系统视图
- `user_tables`
- `all_tables`
- `dba_tables`
- `user_tab_columns`
- `all_tab_columns`
- `v$version`

理解权限边界很重要：
- `user_*`：当前用户拥有的对象
- `all_*`：当前用户可访问的对象
- `dba_*`：高权限视图

## 常见绕过思路
### 1. 语法差异绕过
把 MySQL 习惯改成 Oracle 语法：
- 用 `dual`
- 用 `||` 拼接字符串
- 用 `rownum` 控制条数

### 2. 过滤绕过
重点关注：
- 空格过滤
- 单引号过滤
- 关键字过滤
- 大小写或注释绕过

### 3. 无回显场景
优先看：
- 布尔回显
- 时间回显
- 报错信息
- 带外通道

## 实战排查思路
### 1. 先确认数据库类型
不要把 Oracle 当成 MySQL 直接打。

### 2. 先确认回显能力
区分：
- Union 可用
- 报错可用
- 真假回显
- 时间差回显

### 3. 再切到 Oracle 特有语法
重点替换：
- `limit` -> `rownum`
- 无表查询 -> `dual`
- 拼接方式 -> `||`

## 防御要点
### 1. 参数化查询
这依旧是 Oracle 注入最核心的防御手段。

### 2. 最小权限
数据库账号不应拥有过高权限，更不应开放危险网络包和管理包。

### 3. 关闭详细报错
避免把完整 SQL 异常、对象名、包名、版本信息暴露给前端。

### 4. 统一过滤不是根本方案
不要依赖字符串替换黑名单，应从预编译和 ORM 安全使用入手。

## 速查清单
- 先确认是 Oracle，再切换语法思路
- 先测 `dual`、`user`、`v$version`
- 先尝试 Union，再尝试报错、Bool、时间盲注
- 枚举 `user_*`、`all_*`、`dba_*` 视图
- 关注网络包、XML 函数、权限边界和带外通道

## Reference
- [Oracle 注入指北](https://www.tr0y.wang/2019/04/16/Oracle%E6%B3%A8%E5%85%A5%E6%8C%87%E5%8C%97/)
