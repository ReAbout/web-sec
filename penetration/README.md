# Penetration 笔记索引

本目录用于整理渗透测试过程中常用的操作笔记、命令速查和环境搭建方法。文档统一采用以下结构：

- `概述`：说明适用场景和目标。
- `前置条件`：说明权限、环境或依赖。
- `操作步骤`：给出常用流程和命令。
- `注意事项`：记录容易踩坑的点。
- `参考`：保留外部资料链接。

## 信息收集与扫描

- [PEN-Info.md](./PEN-Info.md)：外网信息收集与目标扩展思路。
- [PEN-Scanner.md](./PEN-Scanner.md)：端口扫描、指纹识别和目录爆破。

## 会话建立与代理转发

- [PEN-ReShell.md](./PEN-ReShell.md)：反弹 Shell、正向 Shell、TTY 升级。
- [PEN-ssh.md](./PEN-ssh.md)：SSH 本地转发、远程转发、动态代理。
- [PEN-Tun2socks.md](./PEN-Tun2socks.md)：Windows 下使用 tun2socks 接管流量。
- [PEN-Openwrt.md](./PEN-Openwrt.md)：OpenWrt 网关代理方案。
- [PEN-Reuse.md](./PEN-Reuse.md)：端口复用和转发思路。

## 凭证与权限提升

- [PEN-GetHash.md](./PEN-GetHash.md)：Windows Hash 与明文凭证获取。
- [PEN-GetHash-Linux.md](./PEN-GetHash-Linux.md)：Linux 凭证与口令窃取思路。
- [PEN-Setuid-Linux.md](./PEN-Setuid-Linux.md)：Linux SUID 提权。
- [PEN-Linux-LPE.md](./PEN-Linux-LPE.md)：Linux 常见本地提权枚举与利用方向。

## 痕迹与运维辅助

- [PEN-LinuxClear.md](./PEN-LinuxClear.md)：Linux 登录痕迹与 History 处理。
- [PEN-WinCmd.md](./PEN-WinCmd.md)：Windows 常用系统与域渗透命令。
- [PEN-MSF.md](./PEN-MSF.md)：Metasploit 与 Meterpreter 常用命令。

## WebShell 与近源专题

- [PEN-Webshell-Question.md](./PEN-Webshell-Question.md)：WebShell 命令执行异常排查。
- [Webshell-Bypass.md](./Webshell-Bypass.md)：WebShell 免杀与流量规避思路速记。
- [PEN-WiFi-Tool.md](./PEN-WiFi-Tool.md)：近源渗透硬件和随身 Wi-Fi 改造。
