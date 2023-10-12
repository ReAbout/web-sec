# Webshell命令不执行问题处理

## Windows平台

### 返回ret=1
利用`aslistcmd`和`ascmd`切换命令执行程序
```
ret=-1> aslistcmd
C:/Windows/System32/cmd.exe            OK
C:/Windows/SysWOW64/cmd.exe            OK
C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe            OK
C:/Windows/SysWOW64/WindowsPowerShell/v1.0/powershell.exe            OK
C:/Windows/System32/WindowsPowerShell/v2.0/powershell.exe            FAIL
C:/Windows/SysWOW64/WindowsPowerShell/v2.0/powershell.exe            FAIL
C:/Windows/System32/WindowsPowerShell/v3.0/powershell.exe            FAIL
C:/Windows/SysWOW64/WindowsPowerShell/v3.0/powershell.exe            FAIL
C:/Windows/System32/WindowsPowerShell/v4.0/powershell.exe            FAIL
C:/Windows/SysWOW64/WindowsPowerShell/v4.0/powershell.exe            FAIL
ret=-1> ascmd C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe
Will execute the command with C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe.
```
