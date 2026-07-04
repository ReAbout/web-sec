# Windows 常用渗透命令速查

## 0x01 概述

本篇用于快速记录 Windows 主机和域环境中常见的信息收集、远程文件操作、远程命令执行与权限切换命令。重点是查用，不展开讲原理。

## 0x02 信息收集

### 1. 本机信息

- 查看系统信息：`systeminfo`
- 查看进程：`tasklist`
- 查看当前工作目录：`chdir`
- 查看网络连接：`netstat -ano`
- 查看本地用户：`net user`
- 查看本地管理员组：`net localgroup administrators`
- 查看指定用户信息：`net user abc`
- 查看已建立共享连接：`net use`
- 查看远程桌面连接历史：`cmdkey /l`
- 查看杀毒软件：`wmic /namespace:\\root\securitycenter2 path antivirusproduct GET displayName,productState,pathToSignedProductExe`
- 查询当前登录用户：`query user`

### 2. 域信息

有时候本地用户查不到完整结果，可以先用 `runas` 切到域用户上下文。

- 查看域：`net view /domain`
- 查看指定域内主机：`net view /domain:abc`
- 查看域用户：`net user /domain`
- 查看域管理员组：`net group "domain admins" /domain`
- 查看本机加入的管理员关系：`net localgroup administrators /domain`
- 查看域控主机：`dsquery server`

## 0x03 远程文件操作

### 1. `net use`

前置条件：

- 目标主机开启 `IPC$` 共享。
- 常用端口为 `445`。

示例目标：

- 主机：`192.168.0.1`
- 用户：`abc`
- 密码：`password`

- 建立空连接：`net use \\192.168.0.1\ipc$ "" /user:""`
- 建立非空连接：`net use \\192.168.0.1\ipc$ "password" /user:"abc"`
- 映射默认共享：`net use z: \\192.168.0.1\c$ "password" /user:"abc"`
- 删除 IPC 连接：`net use \\192.168.0.1\ipc$ /del`
- 删除映射盘：`net use z: /del`

### 2. `net share`

本机开启共享：

```cmd
net share c$=c:
net share d$=d:
net share ipc$
net share admin$
```

### 3. 文件压缩与解压

- 压缩文件：`makecab file`
- 解压文件：`expand src tar.zip`

## 0x04 远程命令执行

### 1. `wmic`

前置条件：

- 目标开启 `Windows Management Instrumentation` 服务。
- 常用端口为 `135`。

执行命令：

```cmd
wmic /node:192.168.0.1 /user:abc /password:password PROCESS call create "cmd /c ver > c:\\test.txt"
```

### 2. `psexec`

一般配合 `net use` 使用，把输出重定向到文件后再读取。

前置条件：

- 目标开启 `ADMIN$` 共享。
- 常用端口为 `445`。

工具：

- https://learn.microsoft.com/zh-cn/sysinternals/downloads/psexec

执行交互式命令：

```cmd
psexec \\192.168.0.1 -u abc -p password cmd
```

### 3. `at`

前置条件：

- 目标启动 `Task Scheduler` 服务。

- 添加任务：`at \\192.168.0.1 23:00 cmd.exe /c "ver > c:\test.txt"`
- 查看任务：`at \\192.168.0.1`
- 删除任务：`at \\192.168.0.1 1 /delete`

### 4. `winrm`

前置条件：

- 目标启动 `WinRM` 服务。
- 常用端口为 `5985`、`5986`。

目标侧快速配置：

```powershell
winrm quickconfig -q
winrm set winrm/config/Client @{TrustedHosts="*"}
```

执行命令：

```cmd
winrs -r:http://192.168.0.1:5985 -u:abc -p:password "whoami /all"
```

## 0x05 权限切换

通过指定用户权限启动命令行：

```cmd
runas /user:abc cmd
```

## 0x06 注意事项

- `wmic`、`at` 在新版本系统里可用性和默认配置差异较大，先验证服务状态。
- `psexec`、`winrm`、共享映射通常都会留下较明显的系统日志。
- 远程命令执行如果没有回显，优先改成重定向到文件再读。

## Ref

- https://chen1sheng.github.io/2020/11/30/%E6%B8%97%E9%80%8F/windows/Windows%E5%9F%9F%E6%B8%97%E9%80%8F%E5%B8%B8%E8%A7%81%E5%91%BD%E4%BB%A4/
- https://www.cnblogs.com/LyShark/p/11344288.html
- https://cloud.tencent.com/developer/article/1180419
