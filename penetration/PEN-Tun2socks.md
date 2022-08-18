# Windows下socks客户端全局代理终极解决方案——tun2socks


需求： 通过socks server相关工具已经开启进入内网的socks5代理，在windows攻击机上想全局走socks5代理，那如何解决？     
解决方案： 用tun2socks 启一个虚拟网卡，该网卡流量都走制定配置好的socks5服务，然后配置路由，全局或者部分ip段指定该网卡为网关。    

### 0x01 准备

1. 要使用 tun2socks，需要创建一个虚拟网卡，首先下载 Tap-windows 并安装：    
下载地址：http://build.openvpn.net/downloads/releases/tap-windows-9.22.1-I602.exe   
2. 下载tun2socks工具     
下载地址：https://github.com/eycorsican/go-tun2socks/releases/download/v1.11.2/tun2socks-windows-4.0-amd64.exe.zip
>PS:采用推荐版本，最新版适配有问题
### 0x02 执行

#### 运行tun2socks
`tun2socks-windows-4.0-amd64.exe -tunAddr 10.0.0.2 -tunGw 10.0.0.1 -proxyType socks -proxyServer 127.0.0.1:1080 -dnsServer 8.8.8.8,8.8.4.4`

>上面命令表示 TUN 接口（虚拟网卡）的 IP 为 10.0.0.2，网关为 10.0.0.1，使用 socks5 代理协议，代理服务器地址是 127.0.0.1:1080，虚拟网卡的 DNS 服务器设为 8.8.8.8,8.8.4.4；其中 127.0.0.1:1080 是一个运行在我路由器上的 socks5 服务器，你需要换成自己的 socks5 服务器。    


#### 添加路由

`route add 172.16.9.0 MASK 255.255.255.0  10.0.0.1`
> 访问172.16.9.0/24 都走tun虚拟网卡，即走socks5流量。     

`route -p add 172.16.9.0 MASK 255.255.255.0  10.0.0.1`
> 添加一条永久路由条目（-p 表示永久路由，重启后不丢失）

#### 更换默认路由
`route change 0.0.0.0 MASK 0.0.0.0  10.0.0.1`

## Ref
- https://tachyondevel.medium.com/%E6%95%99%E7%A8%8B-%E5%9C%A8-windows-%E4%B8%8A%E4%BD%BF%E7%94%A8-tun2socks-%E8%BF%9B%E8%A1%8C%E5%85%A8%E5%B1%80%E4%BB%A3%E7%90%86-aa51869dd0d
