# 域信息收集与 BloodHound

## 0x01 概述
进入域环境后第一件事不是乱打，而是"看清地图"：谁是什么权限、哪台机器有谁登录、从当前位置到域管的最短路径是什么。BloodHound 用图数据库把域内关系可视化，是域渗透的标准侦察工具。

## 0x02 前置条件
- 已有一个域内立足点（普通域用户即可开始收集）
- 本地装好 BloodHound + neo4j（社区版即可）

## 0x03 数据采集
### 1. SharpHound（Windows，C# 采集器）

```powershell
# exe 版全量采集
SharpHound.exe -c All

# 内存加载 ps1 版（少落地）
powershell -ep bypass
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All
```

产出 zip 包，拷回本地导入 BloodHound。

### 2. bloodhound.py（Linux，走 LDAP 远程采集）

```bash
bloodhound-python -u user -p 'password' -d domain.local -ns 192.168.1.10 -c all --zip
```

无需落地 Windows，适合从 Linux 攻击机直接收集；`-ns` 指向域控/DNS。

### 3. 轻量/隐蔽场景
- 只采集关键项：`-c Session,LoggedOn,ACL,ObjectProps` 之类按需组合
- 分次采集降低流量峰值

## 0x04 导入与分析
1. 启动 neo4j，打开 BloodHound，拖入 zip
2. 左上角搜自己的用户名/机器名，右键 **Mark as Owned**
3. 常用内置查询：
   - `Shortest Paths to Domain Admins from Owned Principals`：从已控节点到域管的最短路径
   - `Find Principals with DCSync Rights`：找能直接 DCSync 的账户
   - `Find Kerberoastable Users`：kerberoasting 目标清单（接 [PEN-Kerberos](./PEN-Kerberos.md)）
   - `Find Computers with Unconstrained Delegation`：非约束委派机器
   - `Shortest Paths to High Value Targets`：高价值目标路径
4. 路径边的含义决定打法：`MemberOf`（组嵌套）、`AdminTo`（本地管理员）、`HasSession`（有高权限会话可抓）、`GenericAll/WriteDacl`（ACL 滥用）、`CanRDP` 等

## 0x05 手工收集备选（免 BloodHound 特征）
SharpHound 流量特征明显，EDR 严管环境改手工：

```powershell
# PowerView 常用
Get-DomainUser | select samaccountname
Get-DomainGroupMember "Domain Admins"
Get-DomainComputer | select dnshostname, operatingsystem
Find-LocalAdminAccess        # 扫自己在哪些机器有本地管理员
```

```bash
# Adfind（轻量 LDAP 查询）
Adfind.exe -b "DC=domain,DC=local" -f "objectcategory=person" samaccountname
```

Windows 内建命令也可完成基础侦察，见 [PEN-WinCmd](./PEN-WinCmd.md)。

## 0x06 注意事项
- `Session` 收集（查每台机器谁登录着）噪声最大，隐蔽行动时可先跳过
- BloodHound 数据是"拍照"，域内变化快，打完关键节点后建议重新采集
- neo4j 默认监听本地，别把数据库端口暴露到公共网络
- 采集包里有全量域信息，妥善保管，打完即焚

## 0x07 参考
- [BloodHound](https://github.com/SpecterOps/BloodHound)
- [bloodhound.py](https://github.com/dirkjanm/BloodHound.py)
- [PowerView](https://github.com/PowerShellMafia/PowerSploit)
