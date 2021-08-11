# Iptables 端口复用方法

## 0x01 端口复用

依照源地址区分，让源地址为192.168.10.13访问80端口的流量转发到本地的22端口服务。    
```
iptables -t nat -A PREROUTING -p tcp -s 192.168.10.13 --dport 80 -j REDIRECT --to-port 22
```
查看nat表规则是否生效
```
iptables -t nat -nL

```
## Ref
 - https://www.wangan.com/articles/1050
