# OpenWrt on VMware 网关方案

## 0x01 概述

适用于在本地实验环境中，把 OpenWrt 部署为虚拟网关，让其他虚拟机或宿主机统一通过它上网、做代理或走隧道。

## 0x02 准备

- VMware 虚拟化环境
- OpenWrt x86/x64 镜像
- `qemu-img`
- 至少两块虚拟网卡

## 0x03 安装

### 1. 下载固件：

推荐 https://github.com/DHDAXCW/OpenWRT_x86_x64

### 2. img转vmdk

使用 `qemu-img` 转换镜像格式：

```
qemu-img convert -f raw openwrt.img -O vmdk openwrt.vmdk
```

### 3. VM安装openwrt

vmdk方式虚拟机添加

### 4. 配置openwrt

配置网卡：  
- 正常需要2个网卡，如果缺少，通过虚拟机添加网络适配器（网卡）    
- 一个NAT模式(eth0)，一个主机网络模式(eth1)
- openwrt 可以通过`ifconfig eth1 up`启动

配置网络：
配置wan口和lan口 `vim /etc/config/network`
>注意192.168.111.3根据主机网络模式的网段配置

```
config interface 'lan'
  option type 'bridge'
  option ifname 'eth1'
  option proto 'static'
  option ipaddr '192.168.111.3'
  option netmask '255.255.255.0'
  option ip5assign '60'
config interface 'wan'
  option ifname 'eth0'
  option proto 'dhcp'
```
### 5. 配置其他虚拟机使用OpenWrt来提供网络服务

>其他虚拟机需要通过OpenWrt虚拟机上网时只需要将网卡修改为VMnet1仅主机网络并使用DHCP方式。

需要添加默认网关：    
- Linux:   `route add default gw 192.168.111.3`
- Windows:  高级网络设置修改网关修改即可

### 6. 配置宿主机通过OpenWrt上网

[VMware安装OpenWrt虚拟机让宿主机上网](https://blog.yqxpro.com/2019/10/04/VMware%E5%AE%89%E8%A3%85OpenWrt%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%AE%A9%E5%AE%BF%E4%B8%BB%E6%9C%BA%E4%B8%8A%E7%BD%91/)有图文介绍

## 0x04 代理配置

- VPS安装：v2ray一键脚本： bash <(curl -sL https://s.hijk.art/xray.sh)

- Openwrt插件推荐使用 Passwall，注意要配置全局tcp，udp以及dns

- 客户端检测：可以通过该网站检测IP和DNS  https://whoer.net/

>PS:DNS如果存在泄漏问题：Windows需要通过 `ipconfig /flushdns`刷新下

## 0x05 检查点

- OpenWrt 自己是否已经拿到 IP 并能访问外网。
- `WAN` 与 `LAN` 是否对应到正确网卡。
- 其他虚拟机的默认网关是否真的指向 OpenWrt。
- DNS 请求是否也经过 OpenWrt 或代理插件。

## 0x06 注意事项

- `WAN` 和 `LAN` 网卡不要接反，否则很容易出现只有单向通的情况。
- 先确认 OpenWrt 自己能联网，再让其他虚拟机切网关。
- 代理插件除了 TCP，还要检查 UDP 和 DNS 是否一起接管。

## Ref
- [VMware安装OpenWrt虚拟机让宿主机上网](https://blog.yqxpro.com/2019/10/04/VMware%E5%AE%89%E8%A3%85OpenWrt%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%AE%A9%E5%AE%BF%E4%B8%BB%E6%9C%BA%E4%B8%8A%E7%BD%91/)
