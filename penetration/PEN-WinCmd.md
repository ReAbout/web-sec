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
### 1.2 域信息
- 查看域：`net view /domain`
- 查看abc域中的主机：`net view /domain:abc` 
- 查看域用户 ：`net user /domain`
- 查看域管理员 ： `net group "domain admin" /domain`
- 查看本机域管理员 ：`net localgroup administrators /domain `

## 0x02 远程文件操作
### net use
前置条件：目标主机开启ipc$共享   
目标主机：192.168.0.1 用户名：abc 密码：password
- 建立空连接: `net use \\192.168.0.1\ipc$ "" /user:"" `
- 建立非空连接: `net use \\192.168.0.1\ipc$ "password" /user:"abc" `
 - 映射默认共享: `net use z: \\192.168.0.1\c$ "password" /user:"abc"`
 - 删除一个ipc$连接： `net use \\192.168.0.1\ipc$ /del`
- 删除映射的z盘: `net use z: /del `

## 0x03 远程命令执行
### wmic
前置条件：目标开启 "Windows Management Instrumentation" 服务，端口：135   
目标主机：192.168.0.1 用户名：abc 密码：password   
- 执行命令 ： `wmic /node:192.168.0.1 /user:abc /password:password PROCESS call create "calc.exe"`


## 0x04 权限切换

- 通过abc用户权限启动cmd：
`runas /user:abc cmd`


## ref
- https://chen1sheng.github.io/2020/11/30/%E6%B8%97%E9%80%8F/windows/Windows%E5%9F%9F%E6%B8%97%E9%80%8F%E5%B8%B8%E8%A7%81%E5%91%BD%E4%BB%A4/
- https://www.cnblogs.com/LyShark/p/11344288.html
