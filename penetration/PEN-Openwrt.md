# 全局代理[VMware]：Openwrt on VMware网关方案

## Install

1. 下载固件：

推荐 https://github.com/DHDAXCW/OpenWRT_x86_x64

2. img转vmdk
qemu-img
```
qemu-img convert -f raw openwrt.img -O vmdk openwrt.vmdk
```

3. 如何安装：
配置网卡：  
- 正常需要2个网卡，如果缺少，通过虚拟机添加网络适配器（网卡）    
- 一个NAT模式(eth0)，一个主机共享模式(eth1)
- openwrt 可以通过`ifconfig eth1 up`启动

配置网络：
配置wan口和lan口 `vim /etc/config/network`
```
config interface 'lan'
  option type 'bridge'
  option ifname 'eth1'
  option proto 'static'
  option ipaddr '192.168.111.1'
  option netmask '255.255.255.0'
  option ip5assign '60'
config interface 'wan'
  option ifname 'eth0'
  option proto 'dhcp'
```

- [VMware安装OpenWrt虚拟机让宿主机上网](https://blog.yqxpro.com/2019/10/04/VMware%E5%AE%89%E8%A3%85OpenWrt%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%AE%A9%E5%AE%BF%E4%B8%BB%E6%9C%BA%E4%B8%8A%E7%BD%91/)



## Ref
- [VMware安装OpenWrt虚拟机让宿主机上网](https://blog.yqxpro.com/2019/10/04/VMware%E5%AE%89%E8%A3%85OpenWrt%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%AE%A9%E5%AE%BF%E4%B8%BB%E6%9C%BA%E4%B8%8A%E7%BD%91/)
