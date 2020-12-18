# 前端安全-CSRF
[toc]
## Introduction
跨站请求伪造（Cross-site request forgery）   
跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。   

>XSS与CSRF的区别：   
跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。   


## 防护措施
防御的本质：添加攻击者获取不得的凭证。   
### 同源检测
### CSRF Token
CSRF的另一个特征是，攻击者无法直接窃取到用户的信息（Cookie，Header，网站内容等），仅仅是冒用Cookie中的信息。   
可以要求所有的用户请求都携带一个CSRF攻击者无法获取到的Token。服务器通过校验请求是否携带正确的Token，来把正常的请求和攻击的请求区分开，也可以防范CSRF的攻击。   
### 双重Cookie验证
### Referer

### 特殊的HTTP头字段

## Bypass
### Bypass CSRF Token

* 删除令牌：删除cookie/参数中token，免服务器验证
* 令牌共享：创建两个帐户，替换token看是否可以互相共用；
* 篡改令牌值：有时系统只会检查CSRF令牌的长度；
* 解码CSRF令牌：尝试进行MD5或Base64编码
* 修改请求方法：post改为get
* 窃取token：重定向、XSS、web缓存欺骗、clickjacking等都可能导致token泄露

### Referer
如果是通过正则判断，利用正则的特点。   
购买个包含目标域名的域名，`http://baidu.com.reabout.com`   
通过参数的形式`http: //attack.com?http://baidu.com`   



- Ref[]:[前端安全系列（二）：如何防止CSRF攻击？](https://tech.meituan.com/2018/10/11/fe-security-csrf.html)  

- [跨站请求伪造（CSRF）挖掘技巧及实战案例全汇总](https://cloud.tencent.com/developer/article/1516424)