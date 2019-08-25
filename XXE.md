# EXP手册-XXE(XML External Entity Injection)
## 基础知识
XXE(XML External Entity),即是XML外部实体注入攻击.漏洞是在对不安全的外部实体数据进行处理时引发的安全问题。
关键在DTD的引用。
DTD(The document type definition)，即是文档类型定义，可定义合法的XML文档构建模块。它使用一系列合法的元素来定义文档的结构。DTD 可被成行地声明于 XML 文档中，也可作为一个外部引用。
### 

