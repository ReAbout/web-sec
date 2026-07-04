# Windows 下使用 tun2socks 接管流量

## 0x01 概述

适用于已经拿到一个可用的 `SOCKS5` 代理，希望让 Windows 攻击机上的指定网段或全局流量都走该代理的场景。

核心思路：

1. 用 `tun2socks` 创建虚拟网卡。
2. 把虚拟网卡流量转进 SOCKS5 代理。
3. 用路由控制哪些目标网段走这张虚拟网卡。

## 0x02 准备

1. 要使用 tun2socks，需要创建一个虚拟网卡，首先下载 Tap-windows 并安装：    
下载地址：http://build.openvpn.net/downloads/releases/tap-windows-9.22.1-I602.exe   
2. 下载tun2socks工具     
下载地址：https://github.com/eycorsican/go-tun2socks/releases/download/v1.11.2/tun2socks-windows-4.0-amd64.exe.zip
>PS:采用推荐版本，最新版适配有问题

## 0x03 执行

#### 运行tun2socks
`tun2socks-windows-4.0-amd64.exe -tunAddr 10.0.0.2 -tunGw 10.0.0.1 -proxyType socks -proxyServer 127.0.0.1:1080 -dnsServer 8.8.8.8,8.8.4.4`

>上面命令表示 TUN 接口（虚拟网卡）的 IP 为 10.0.0.2，网关为 10.0.0.1，使用 socks5 代理协议，代理服务器地址是 127.0.0.1:1080，虚拟网卡的 DNS 服务器设为 8.8.8.8,8.8.4.4；其中 127.0.0.1:1080 是一个运行在我路由器上的 socks5 服务器，你需要换成自己的 socks5 服务器。    


#### 添加路由

`route add 172.16.9.0 MASK 255.255.255.0  10.0.0.1`
> 访问172.16.9.0/24 都走tun虚拟网卡，即走socks5流量。     

`route -p add 172.16.9.0 MASK 255.255.255.0  10.0.0.1`
> 添加一条永久路由条目（-p 表示永久路由，重启后不丢失）

#### 全局代理
`route change 0.0.0.0 MASK 0.0.0.0  10.0.0.1 metric 6`
>更改默认路由
`route add x.x.x.x 192.168.101.1 metric 5`
>还要加一条路由让去我们代理服务器的流量发到原来的网关（192.168.101.1）

## 0x04 注意事项

- 先确认本机到 SOCKS5 服务器本身的路由不要被默认路由覆盖。
- 如果出现 DNS 泄漏，要同时检查系统 DNS、浏览器 DoH 和代理软件设置。
- 全局代理前先验证单网段代理可用，再切默认路由。

## Ref
- https://tachyondevel.medium.com/%E6%95%99%E7%A8%8B-%E5%9C%A8-windows-%E4%B8%8A%E4%BD%BF%E7%94%A8-tun2socks-%E8%BF%9B%E8%A1%8C%E5%85%A8%E5%B1%80%E4%BB%A3%E7%90%86-aa51869dd0d
