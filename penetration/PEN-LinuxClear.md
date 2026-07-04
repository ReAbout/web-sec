# Linux 痕迹清理

## 0x00 概述

主要记录 Linux 常见登录痕迹和命令历史相关文件的位置，以及手工清理或篡改的方法。适用于已经获得主机权限后，对登录记录和交互痕迹做最基础检查。

## 0x01 ssh登录日志

### 日志文件位置及命令
 | 命令	 | 日志文件	 | 功能 | 
 | ---- | ---- |---- |
 |w,who|/var/run/utmp|记录当前正在登录系统的用户信息，uptime记录系统启动时间|
 | last | 	/var/log/wtmp	 | 所有成功登录/登出的历史记录 | 
 | lastb | 	/var/log/btmp | 	登录失败尝试 | 
 | lastlog	 | /var/log/lastlog	 | 最近登录记录 | 

这些日志都是以二进制形式存储。   

### 清除方法
方案1，直接清空：  
```
# > /var/log/utmp
# > /var/log/wtmp
# > /var/log/btmp
# > /var/log/lastlog
```
方案2，使用脚本：

- [logtamper.py](./logtamper.py)
```
躲避管理员w查看(w)
python logtamper.py -m 1 -u root -i 192.168.0.188

清除指定ip的登录日志(lastb)
python logtamper.py -m 2 -u root -i 192.168.0.188

修改上次登录时间地点(lastlog)
python logtamper.py -m 3 -u root -i 192.168.0.188 -t tty1 -d 2014:05:28:10:11:12
```
### 不记录history
```
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG
export HISTFILE=/dev/null
export HISTSIZE=0
export HISTFILESIZE=0
```

## 0x02 注意事项

- 不同发行版的日志路径、日志轮转和审计配置可能不同。
- 直接清空日志虽然简单，但特征很明显，容易被对比发现。
- 如果系统启用了 `auditd`、集中日志或堡垒机审计，本地处理并不能覆盖全部痕迹。
