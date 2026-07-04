# 扫描器专题

## 0x01 概述

扫描阶段的目标不是“把所有工具都跑一遍”，而是尽快回答三个问题：

1. 目标开放了哪些端口和服务。
2. 服务对应的协议、产品和版本是什么。
3. 哪些入口值得继续做漏洞验证和目录枚举。

## 0x02 常用流程

### 1. 主机发现

适合先确认存活主机，再决定后续扫描范围。

```bash
fping -asg 192.168.1.0/24
```

```bash
nmap -sn 192.168.1.0/24
```

### 2. 端口发现

#### `masscan`

优点是快，适合大网段粗扫。

```bash
masscan 192.168.1.0/24 -p1-65535 --rate 5000
```

#### `naabu`

适合快速发现开放端口，并和后续 Web 指纹工具配合。

```bash
naabu -host 192.168.1.10 -p top-1000
```

### 3. 服务识别

#### `nmap`

```bash
nmap -sV -sC -O -Pn 192.168.1.10
```

常用参数：

- `-sV`：服务版本识别。
- `-sC`：执行默认脚本。
- `-O`：系统识别。
- `-Pn`：跳过主机发现。

### 4. Web 资产识别

#### `httpx`

```bash
httpx -u http://192.168.1.10 -title -tech-detect -status-code
```

适合批量确认：

- 标题
- 状态码
- 中间件
- 是否存在跳转

### 5. 目录与接口枚举

#### `ffuf`

```bash
ffuf -u http://192.168.1.10/FUZZ -w wordlists.txt -mc all -fc 404
```

常见使用方式：

- 爆破目录：`/FUZZ`
- 爆破文件：`/admin/FUZZ.php`
- 爆破参数值：`/api?id=FUZZ`

### 6. 漏洞模板扫描

#### `nuclei`

```bash
nuclei -u http://192.168.1.10 -severity critical,high,medium
```

适合在确认指纹后做高效验证，但要注意：

- 模板版本是否最新。
- 请求特征是否容易触发 WAF。
- 不要对不稳定目标直接跑高危 POC。

## 0x03 Web 扫描建议

### 1. 先轻量识别，再做深度枚举

推荐顺序：

1. `httpx`
2. `nmap -sV`
3. `ffuf`
4. `nuclei`

### 2. 关注这些高价值信号

- 后台路径：`/admin`、`/manage`、`/console`
- 调试信息：报错页、目录列表、Swagger、Actuator
- 组件路径：`/jmx-console`、`/manager/html`
- 默认页面：路由器、NAS、监控系统、OA、CMS

## 0x04 内网扫描建议

- 优先控速，避免广播式高频扫描打爆交换机或告警设备。
- 先确定关键网段，再按业务优先级细扫。
- 对域环境先看 `445`、`135`、`139`、`88`、`389`、`3389`、`5985`。

示例：

```bash
nmap -Pn -p 445,135,139,88,389,3389,5985 192.168.10.0/24
```

## 0x05 常用命令速查

```bash
# Top 端口扫描
nmap --top-ports 1000 192.168.1.10

# 全端口 + 服务识别
nmap -p- -sV 192.168.1.10

# Web 标题和指纹
httpx -l urls.txt -title -tech-detect

# 目录爆破
ffuf -u http://target/FUZZ -w dict.txt -fc 404

# 模板扫描
nuclei -l urls.txt -severity high,critical
```

## 0x06 注意事项

- 扫描速度要和网络环境匹配，尤其是内网与公网弱带宽主机。
- 不要把漏洞验证和端口发现混在同一阶段，便于回溯问题。
- 对 Web 目标先做去重，避免对同一资产重复打点。

## Ref

- https://nmap.org/book/man.html
- https://github.com/projectdiscovery/naabu
- https://github.com/projectdiscovery/httpx
- https://github.com/projectdiscovery/nuclei
- https://github.com/ffuf/ffuf
