# 端口复用与转发

## 0x01 概述

端口复用适用于目标机器上某个高价值端口已经被业务占用，但你希望复用该端口对特定来源或特定流量做转发。常见场景：

- 只对你的来源 IP 复用 `80` / `443`。
- 把入口流量重定向到本地其他服务。
- 在不新增明显监听端口的情况下维持访问路径。

## 0x02 `iptables` 按来源地址复用

依照源地址区分，让来源为 `192.168.10.13` 访问 `80` 端口的流量转发到本地 `22` 端口：

```bash
iptables -t nat -A PREROUTING -p tcp -s 192.168.10.13 --dport 80 -j REDIRECT --to-port 22
```

查看规则是否生效：

```bash
iptables -t nat -nL
iptables -t nat -vnL PREROUTING
```

删除规则：

```bash
iptables -t nat -D PREROUTING -p tcp -s 192.168.10.13 --dport 80 -j REDIRECT --to-port 22
```

## 0x03 DNAT 转发到其他主机

如果要把入口端口转发到内网其他主机，而不是本机服务，可以使用 DNAT：

```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.20:80
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.20 --dport 80 -j MASQUERADE
```

适合做跳转入口或临时暴露内网服务。

## 0x04 使用 `socat` 做本地复用

当系统不方便直接改防火墙时，也可以用用户态工具把流量转到其他端口：

```bash
socat TCP-LISTEN:8080,fork TCP:127.0.0.1:22
```

如果要对外暴露 TLS 前的明文服务，这种方式更直观，但痕迹也更明显。

## 0x05 检查点

- 目标内核是否开启了转发能力。
- 本机防火墙规则顺序是否会覆盖新规则。
- 业务端口是否已经绑定到 `127.0.0.1` 或特定网卡。
- 入口流量是否经过上层代理、CDN 或 WAF。

## 0x06 注意事项

- 端口复用的核心不是“隐藏”，而是“减少新增暴露面”。
- 涉及生产业务端口时，要先确认不会影响原有服务。
- 修改 NAT 规则后，记得同步检查回包路径。

## Ref

- https://book.hacktricks.wiki/en/generic-hacking/tunneling-and-port-forwarding.html
- https://www.wangan.com/articles/1050
