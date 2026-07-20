# 越权漏洞（IDOR / Broken Access Control）

## 一句话理解
越权是指服务端只验证了"你有没有登录"，却没有验证"这个资源是不是你的、这个操作你配不配"，导致低权限用户可以读写他人数据或调用高权限功能。

## 核心原理
- 认证（Authentication）解决"你是谁"，授权（Authorization）解决"你能做什么"
- 越权的本质是服务端把"能访问接口"等同于"有权操作对象"，缺少对象级（object-level）或功能级（function-level）的归属校验
- 前端隐藏按钮、隐藏菜单、不暴露链接都不是防护，只是"藏"；服务端不校验，直接构造请求即可命中

## 三种形态
### 1. 水平越权
同权限用户之间互相访问。例如用户 A 把 `id=1001` 改成 `id=1002`，看到用户 B 的订单。

### 2. 垂直越权
低权限用户使用高权限功能。例如普通用户直接请求管理员接口 `POST /admin/addUser`。

### 3. IDOR
IDOR（Insecure Direct Object Reference，不安全的直接对象引用）是水平越权的典型实现：请求参数直接暴露数据库主键、文件名、订单号等对象引用，服务端未校验归属。OWASP API Security Top 1（BOLA）本质上就是它。

## 成立条件
通常需要满足：
- 请求中存在标识资源的参数（`id`、`uid`、`orderNo`、`fileId`、`doc`）
- 服务端仅校验登录态，不校验参数指向的对象归属当前会话
- 攻击者拥有合法低权限账号（垂直越权时甚至不需要）

## 常见攻击面
- RESTful 接口：`/api/user/{id}`、`/api/order/{orderNo}`
- 查询参数：`?uid=1`、`?file=report.pdf`
- JSON Body：`{"userId": 1001}`、`{"role": "admin"}`（注册时提权）
- 批量接口：`/api/export?ids=1,2,3`，一次拖出大量数据
- 文件下载、预览、导出、分享链接
- 隐藏接口：前端没有入口但接口真实存在（从 JS、Swagger、历史包里翻）
- 修改类操作（POST/PUT/DELETE）越权危害远大于查询类

## 快速判断
### 1. 双账号对比法（推荐）
- 注册/准备两个账号 A、B
- 用 A 的操作抓包，把会话换成 B（Cookie/Token 替换），资源参数保持指向 A 的对象
- 如果 B 能读到/改到 A 的数据，水平越权成立

### 2. 参数递增减
数字型 id 直接 `+1/-1` 遍历，观察返回数据是否属于他人。

### 3. 观察点
- 响应数据是否随参数变化而变化，与会话无关
- 服务端返回的报错是否泄露对象存在性（存在/不存在两种报错可枚举）

## 常见利用技巧
- 参数位置全测一遍：URL 路径、Query、Body 表单、JSON、Cookie、Header（`X-User-Id`）
- JSON 嵌套与数组：`{"user":{"id":2}}`、`id=1&id=2`（参数污染）
- 请求方法篡改：`GET` 拒绝后试 `POST/PUT/PATCH`
- GUID/UUID 不可遍历，但可以结合信息泄露（评论、分享、列表接口）拿到他人 UUID 后再越权
- 越权 + CSRF / 越权 + XSS 可组合放大危害

## 与相邻概念的区别
| 概念 | 区别 |
| --- | --- |
| 未授权访问 | 完全不需要登录态即可访问，越权是"有登录态但权限边界失守" |
| CSRF | 借用受害者的身份发起请求；越权是攻击者用自己的身份访问不该访问的对象 |
| 信息泄露 | 越权读到的数据本身常构成信息泄露，但越权强调的是"校验缺失" |

## 防御要点
- 服务端对每个请求做对象归属校验（`resource.ownerId == session.userId`）
- 功能级接口做角色/权限校验，不依赖前端隐藏
- 优先使用间接引用：会话内维护用户可访问对象列表，参数只传序号
- 不可猜测的 ID（UUID）只能降低枚举效率，不能替代鉴权
- 失败时返回统一提示，避免泄露对象存在性

## 参考
- [OWASP Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [PortSwigger Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [OWASP API Security Top 10 - BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
