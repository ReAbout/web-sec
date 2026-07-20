# 业务逻辑漏洞利用

## 一句话理解
代码本身没有语法错误、没有注入点，但业务流程的"设计假设"被攻击者违反：跳步骤、改金额、并发刷单、重置他人密码——防的是注入，输的是脑子。

## 核心特点
- 自动化扫描器基本扫不出来，靠人工理解业务
- 危害直观（钱、账号、数据），SRC 与实战价值高
- 原理层分析见 [VUL-Logic](../vul/VUL-Logic.md)，本篇聚焦测试手法

## 测试方法论
1. 画出业务流程：注册 -> 登录 -> 操作 -> 支付/提交 -> 回调/完成
2. 对每个环节问五个问题：
   - 参数能不能改？（金额、数量、身份、状态）
   - 步骤能不能跳？（直接请求第 3 步的接口）
   - 操作能不能重？（重复提交、重放、并发）
   - 校验在哪端？（前端 JS 校验 = 没校验）
   - 失败能不能骗？（改响应包、伪造回调）
3. 抓全链路的包，把每个参数、每个接口都过一遍

## 常见类型与手法
### 1. 条件竞争（Race Condition）
同一请求并发多次，服务端校验与扣减之间存在时间窗：
- 超领优惠券、超限提现、积分翻倍、多次投票
- 工具：Burp **Turbo Intruder**（`race.py` 模板）、Burp 2023+ 的 "Send group in parallel"（HTTP/2 single-packet attack，时窗最小）

```python
# Turbo Intruder 竞争模板思路：同一连接/同一时刻齐发 N 个相同请求
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=30, requestsPerConnection=30)
    for i in range(30):
        engine.queue(target.req)
```

### 2. 验证码逻辑
- 验证码可复用（不销毁）、长期有效
- 校验在前端：删掉验证码参数照样过
- 验证码与请求不绑定（拿 A 的码验 B 的请求）
- 响应包返回验证码、验证码复杂度低可爆破

### 3. 短信/邮件轰炸
- 无频率限制、无每日上限 -> 轰炸消耗短信费/骚扰
- 频率限制在前端或用 Cookie 计数 -> 并发或清 Cookie 绕过
- 手机号参数可传数组/逗号分隔 -> 一信多发

### 4. 任意密码重置
- 重置 token 可预测（时间戳、自增、弱随机，见 [VUL-Crypto](../vul/VUL-Crypto.md)）
- token 与用户不绑定：用自己的 token 改别人的密码
- token 长期有效、用后不失效
- **Host 头投毒**：重置邮件里的链接域名取自 `Host` 头，受害者点击后 token 落到攻击者域名
- 改响应包：验证码校验接口返回 `{"code":1}` 改成 `{"code":0}`

### 5. 支付逻辑
- 金额、单价、运费篡改（改包直接改）
- 数量为负、小数为负 -> 金额变负，下单倒赚钱
- 整数溢出：数量设大数使总价溢出变小
- 多币种/单位混淆：分与元、汇率转换漏洞
- 伪造支付回调：直接请求回调接口或篡改回调参数（签名校验缺失）
- 优惠券叠加、折扣码重复使用（配合条件竞争）

### 6. 登录/认证逻辑
- 响应篡改：登录失败返回 `{"success":false}`，改成 `true` 后端却信了前端跳转（前端鉴权）
- 越权登录：登录参数里 `role=admin`、`isAdmin=1` 一并提交
- OTP/二步验证可跳过：直接请求第三步接口

### 7. 越权类
水平/垂直越权是逻辑漏洞的最大分支，单独成篇：[EXP-IDOR](./EXP-IDOR.md)。

## 工具
- Burp Suite：Repeater 改包、Turbo Intruder 并发
- [race-the-web](https://github.com/TheHackerDev/race-the-web)：竞争测试
- 脚本化：Python requests + ThreadPool（自造并发）

## 防御要点
- 关键操作幂等：唯一约束、去重 token、乐观锁/悲观锁
- 服务端重新计算金额、数量、权限，不信任任何客户端值
- 验证码一次一毁、与用户/操作绑定、短时效
- 回调接口验签、校验来源、异步对账
- 限流：IP + 账号 + 设备多维频率限制

## 参考
- [PortSwigger - Business logic vulnerabilities](https://portswigger.net/web-security/logic-flaws)
- [PortSwigger - Race conditions](https://portswigger.net/web-security/race-conditions)
