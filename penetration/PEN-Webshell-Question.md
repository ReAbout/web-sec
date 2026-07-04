# WebShell 命令执行异常排查

## 0x01 概述

WebShell 能连上但命令不执行，通常不是“没有权限”这么简单，更多是以下问题：

- 调用的解释器路径不对。
- Web 容器权限不足。
- 被安全策略、杀软或函数禁用拦截。
- 当前目录、编码或重定向方式有问题。

## 0x02 Windows 平台

### 1. 返回 `ret=1` 或执行失败

先切换命令解释器，排除当前调用程序不可用的问题。

```text
ret=-1> aslistcmd
C:/Windows/System32/cmd.exe            OK
C:/Windows/SysWOW64/cmd.exe            OK
C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe            OK
C:/Windows/SysWOW64/WindowsPowerShell/v1.0/powershell.exe            OK
C:/Windows/System32/WindowsPowerShell/v2.0/powershell.exe            FAIL
C:/Windows/SysWOW64/WindowsPowerShell/v2.0/powershell.exe            FAIL
ret=-1> ascmd C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe
Will execute the command with C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe.
```

### 2. 常见原因

- `cmd.exe` 被替换或不可调用。
- PowerShell 版本存在兼容问题。
- Web 服务用户无执行权限。
- 杀软拦截了子进程创建。
- 输出被编码或重定向吃掉。

### 3. 快速检查项

先验证最短命令：

```cmd
whoami
ver
ipconfig
```

再验证重定向是否正常：

```cmd
whoami > C:\Windows\Temp\whoami.txt
```

如果命令能落文件，但 WebShell 无回显，问题通常在回显链路而不是命令执行本身。

## 0x03 Linux 平台

### 1. 常见原因

- `sh` / `bash` 路径不一致。
- PHP 禁用了 `system`、`exec`、`shell_exec` 等函数。
- Web 服务用户无目录或命令执行权限。
- SELinux / AppArmor 限制。

### 2. 快速检查项

```bash
whoami
id
pwd
/bin/sh -c 'id'
```

如果 `system("id")` 失败，但 `echo` 正常，优先检查禁用函数和解释器路径。

## 0x04 排查顺序

1. 先确认命令解释器路径。
2. 再确认最短命令是否执行。
3. 再看是否有文件写入能力。
4. 最后再查安全策略和编码问题。

## 0x05 注意事项

- 命令执行问题和“没有回显”是两回事，要分开判断。
- Windows 下优先确认 `cmd.exe` 与 PowerShell 版本。
- Linux 下优先确认 `disable_functions`、SELinux、当前用户权限。
