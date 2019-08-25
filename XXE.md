# EXP手册-XXE(XML External Entity Injection)
## 基础知识
XXE(XML External Entity),即是XML外部实体注入攻击.漏洞是在对不安全的外部实体数据进行处理时引发的安全问题。   
关键在DTD的引用。   
DTD(The document type definition)，即是文档类型定义，可定义合法的XML文档构建模块。它使用一系列合法的元素来定义文档的结构。DTD 可被成行地声明于 XML 文档中，也可作为一个外部引用。   
前提条件：允许外部实体引用。   
## XXE漏洞利用
按服务端语言有PHP、JAVA、JAVA(Android)一般解析函数都是默认不开启的。
漏洞利用方法分：
- 回显读取文件   
- 不带回显读取文件    
- 报错回显读取文件
- 远程执行   

### Reference
[一篇文章带你深入理解漏洞之 XXE 漏洞](https://xz.aliyun.com/t/3357)   
[Exploiting XXE with local DTD files](https://mohemiv.com/all/exploiting-xxe-with-local-dtd-files/)   
