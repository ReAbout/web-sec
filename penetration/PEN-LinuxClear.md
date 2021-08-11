# Linux 痕迹清理

## 0x01 ssh登录日志

### 日志文件位置及命令
 | 命令	 | 日志文件	 | 功能 | 
 | ---- | ---- |---- |
 | last | 	/var/log/wtmp	 | 所有成功登录/登出的历史记录 | 
 | lastb | 	/var/log/btmp | 	登录失败尝试 | 
 | lastlog	 | /var/log/lastlog	 | 最近登录记录 | 

这些日志都是以二进制形式存储。   

### 清除方法
方案1，直接清空：  
```
# > /var/log/wtmp
# > /var/log/btmp
# > /var/log/lastlog
```
方案2，使用脚本：

- [logtamper.py](./logtamper.py)

