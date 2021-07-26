# WEB 安全手册 Index
包括个人对漏洞理解，漏洞利用和渗透测试的整理，也收录了他人相关的知识的总结和工具的推荐。    

## 0x01 漏洞理解篇(Vulnerability)
### 1.1 前端
> 同源策略 & CSP & JOSNP
- [跨域安全](https://github.com/ReAbout/web-exp/blob/master/vul/VUL-CrossDomain.md)
### 1.2 后端逻辑
- [错综复杂的后端逻辑及安全]()【待整理】
## 0x02 漏洞利用篇(Exploit)
### 2.1前端安全
> XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任   
- [Cross Site Scripting (XSS)](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-XSS.md)
- [Client-side request forgery (CSRF)](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-CSRF.md)
###  2.2 SQL注入&数据库漏洞利用
- [SQL injection - MySQL](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-SQLi-MySQL.md)
- [SQL injection - 信息外带(OOB)](./web-sec/exp/EXP-SQLi-OOB.md)
### 2.3 模板注入 Server Side Template Injection (SSTI)
> MVC架构中，模板参数恶意输入产生的安全问题
- [SSTI -Python](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-SSTI.md)
### 2.4 表达式注入 

### 2.5 命令注入&执行

### 2.6 Server-side request forgery (SSRF)
- [SSRF](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-SSRF.md)
### 2.7 XML External Entity (XXE) 
- [XXE](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-XXE.md)
### 2.8 反序列化漏洞
>php,java只能序列化数据，python可以序列化代码。   
- [反序列化漏洞-PHP](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-PHP-Unserialize.md)
### 2.9 PHP-特性漏洞
- [包含漏洞-PHP](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-Include-PHP.md)
### 2.10 NodeJs-特性漏洞
- [Node.js 原型链污染](https://github.com/ReAbout/web-exp/blob/master/exp/EXP-nodejs-proto.md)
### 2.11 Other
- [DNS rebinding 攻击]()[待更新]
## 0x03 代码审计篇(Audit)
### 3.1 PHP
- [PHP调试环境的搭建](https://github.com/ReAbout/web-exp/blob/master/audit/AUD-PHP-Debug.md)
- [PHP代码审计@bowu678](https://github.com/bowu678/php_bugs)
### 3.2 JAVA
- [Java代码审计@cn-panda](https://github.com/cn-panda/JavaCodeAudit)

## 0x04 渗透篇(Penetration)
### 4.1 网络预置
- [信息收集思路](https://github.com/ReAbout/web-exp/blob/master/penetration/PEN-Info.md)
- [**[Tool]** 移动端信息收集工具 AppInfoScanner](https://github.com/kelvinBen/AppInfoScanner)
### 4.2 外网突破(exp)

#### 4.2.1 漏洞验证（扫描器）
> 工欲其善必先利器
##### 4.2.1.1 主动式
 - [**[Tool]** AWVS 14 Docker版](https://hub.docker.com/r/secfa/docker-awvs)
 - [**[Tool]** 长亭的扫描器 Xray](https://github.com/chaitin/xray)   
 - [**[Tool]** Vulmap 是一款 web 漏洞扫描和验证工具, 可对 webapps 进行漏洞扫描, 并且具备漏洞利用功能, 目前支持的 webapps 包括 activemq, flink, shiro, solr, struts2, tomcat, unomi, drupal, elasticsearch, fastjson, jenkins, nexus, weblogic, jboss, spring, thinkphp](https://github.com/zhzyker/vulmap)  
##### 4.2.1.2 被动式
>将Burpusuite打造成一个被动式扫描器   
- [**[Tool]** BurpSutie 插件集合@Mr-xn](https://github.com/Mr-xn/BurpSuite-collections)  

#### 4.2.2漏洞利用
- [漏洞索引表]()【待整理】
- [红队中易被攻击的一些重点系统漏洞整理@r0eXpeR](https://github.com/r0eXpeR/redteam_vul)
- [织梦全版本漏洞扫描@lengjibo](https://github.com/lengjibo/dedecmscan)
- [**[Tool]** Struts2-Scan](https://github.com/HatBoy/Struts2-Scan)
- [**[Tool]** shiro反序列化漏洞综合利用](https://github.com/j1anFen/shiro_attack)


### 4.3 权限获取&提升
#### 4.3.1 Win
- [Windows 认证凭证获取]()  
- [**[Tool]** Windows 认证凭证提取神器](https://github.com/gentilkiwi/mimikatz) 
- [Windows提权漏洞集合@SecWiki](https://github.com/SecWiki/windows-kernel-exploits)
#### 4.3.2 Linux
- [linux提权漏洞集合@SecWiki](https://github.com/SecWiki/linux-kernel-exploits)
### 4.4 后门&维持会话
#### 4.4.1 Shell会话
- [反弹shell Linux&Win](./penetration/PEN-ReShell.md)
#### 4.4.2 Webshell
- [**[Tool]** WebShell管理工具 蚁剑](https://github.com/AntSwordProject/AntSword-Loader)
- [**[Tool]** WebShell管理工具 冰蝎](https://github.com/rebeyond/Behinder)
#### 4.4.3 PC & Server
- [**[Tool]** Cobalt Strike ]()
#### 4.4.4 Mobile (Android & ios)  
### 4.5 隧道&代理
- [ssh 端口转发&开socks5](./penetration/PEN-ssh.md)
- [**[Tool]** 反向端口转发工具 FRP](https://github.com/fatedier/frp)
- [**[Tool]** 内网多级代理工具 Venom](https://github.com/Dliv3/Venom/releases)
- [**[Tool]** Windows下代理客户端工具 Proxifier](https://www.proxifier.com/)
### 4.6 后渗透

####  4.6.1 轻量级扫描工具
- [**[Tool]** fscan](https://github.com/shadow1ng/fscan)
#### 4.6.2 渗透框架
- [**[Tool]** 后渗透利用神器 Metasploit](https://www.metasploit.com/)
- [**[Tool]** 内网横向拓展系统 InScan](https://github.com/inbug-team/InScan)
#### 4.6.3 域渗透
- [域渗透@uknowsec](https://github.com/uknowsec/Active-Directory-Pentest-Notes)
### 4.7 反溯源 
### 4.8 协同
- [HackMD markdown协同工具(Docker版)](https://hackmd.io/c/codimd-documentation/%2Fs%2Fcodimd-docker-deployment)


