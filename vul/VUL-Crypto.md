# Web 密码学误用（原理篇）

## 定位
本篇不讲密码算法本身的数学原理（那是 Crypto 方向的事），只讲 **Web 场景里把密码学用错的标准姿势**。每种误用都对应真实漏洞链，利用细节指向 exp/ 篇目。

## 编码 != 加密
- Base64、URL 编码、hex 只是编码，任何人可逆
- 常见幻觉："把数据 Base64 一下放 Cookie 就安全了"——攻击者解码改完再编码即可
- JWT payload 同理：看得懂，就看得到（见 [EXP-JWT](../exp/EXP-JWT.md)）

## 对称加密的误用
### 1. ECB 模式：分组重排
- ECB 每个明文块独立加密，相同明文块 -> 相同密文块
- 攻击者可以**切块重排**：把密文里"普通用户"块替换成另一个账号的"管理员"块
- 识别特征：密文长度是块整数倍，修改明文前缀观察密文重复段

### 2. CBC 比特翻转（Bit-flipping）
- CBC 解密时，篡改第 N 块密文的某字节，会可控地翻转第 N+1 块明文的对应字节
- 经典场景：Cookie 密文里 `role=0` 翻成 `role=1`（字节异或差值算好再改）
- 前提：知道部分明文结构，这在 Cookie/Token 场景几乎总是成立

### 3. CBC Padding Oracle
- 服务端对"填充是否正确"给出可区分响应（报错不同/时间不同）
- 攻击者逐字节解密任意密文，甚至加密任意明文
- 实战案例：**Shiro-721**（rememberMe 的 AES-CBC + 报错差异），工具见 README 4.2.2.2；**ASP.NET ViewState** 同理
- 工具：[padbuster](https://github.com/AonCyberLabs/PadBuster)

## 哈希的误用
### 1. 哈希长度扩展（Length Extension）
- MD5/SHA1/SHA256 等 Merkle-Damgard 结构哈希：知道 `H(secret + data)` 和 secret 长度，就能算出 `H(secret + data + padding + append)`
- 经典场景：签名设计为 `sign = md5(secret + params)`，攻击者不知道 secret 也能追加参数并伪造合法签名
- 工具：hashpump、[hash_extender](https://github.com/iagox86/hash_extender)
- 正确姿势：用 HMAC 而不是裸哈希拼接

### 2. 弱哈希存口令
- MD5/SHA1 无盐存口令 -> 彩虹表直接查
- 见 [VUL-Auth-Session](./VUL-Auth-Session.md) 凭证存储节

### 3. 加密当签名 / 签名当加密
- 只加密不验签 -> 可篡改（CBC 翻转、ECB 重排）
- 正确姿势：Encrypt-then-MAC，或直接用 AEAD（AES-GCM）

## 随机数误用
- **伪随机当安全随机**：`rand()`、`mt_rand()`、时间戳做种子 -> 可预测
- 高危场景：密码重置 token、Session ID、CSRF token、订单号
- 经典案例：PHP `mt_rand` 种子泄露后全序列预测；Java `java.util.Random` 同理
- 正确姿势：`/dev/urandom`、`random_bytes()`、`SecureRandom`、`secrets` 模块

## 密钥管理失误
- **硬编码密钥**：代码/配置文件里的密钥 -> 源码泄露即全线失守（Shiro-550 就是密钥硬编码在公开组件里）
- **默认密钥**：框架默认值不改（JWT `secret`、Django `SECRET_KEY` 示例值）
- **密钥复用**：加密和签名用同一把钥匙，一个环节泄露牵连全部

## 比较与时序
- 用 `==` 比较 token/签名 -> 逐字节短路比较泄露时序，理论可逐位爆破
- 防御侧用 `hash_equals()`、`hmac.compare_digest()` 恒定时间比较
- 实战意义有限（噪声大），但 CTF 偶见

## 速查映射表
| 你看到的 | 应该怀疑 | 工具/方向 |
| --- | --- | --- |
| Cookie/参数是分组整数倍的密文 | ECB 重排、CBC 翻转、Padding Oracle | padbuster、手算异或 |
| `sign=md5(...)` 形式的签名 | 长度扩展 | hashpump、hash_extender |
| token 看起来像时间戳/自增 | 弱随机预测 | 收集样本建模 |
| 报错区分"填充错误" | Padding Oracle | padbuster |
| 源码里躺着密钥 | 硬编码/默认密钥 | 接 [EXP-InfoLeak](../exp/EXP-InfoLeak.md) |

## 参考
- [OWASP Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
