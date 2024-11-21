# 绕过过高版本Jdk的限制进行Jndi注入利用

## 0x00限制成因

除了RMI服务之外，JNDI还可以对接LDAP服务，LDAP也能返回JNDI Reference对象，利用过程与上面RMI Reference基本一致，只是lookup()中的URL为一个LDAP地址：ldap://xxx/xxx，由攻击者控制的LDAP服务端返回一个恶意的JNDI Reference对象。并且LDAP服务的Reference远程加载Factory类不受上一点中 com.sun.jndi.rmi.object.trustURLCodebase、com.sun.jndi.cosnaming.object.trustURLCodebase等属性的限制，所以适用范围更广。

### 1. trustURLCodebase = false （CVE-2018-3149）
2018年10月，高版本JDK在RMI和LDAP的trustURLCodebase都做了限制，从默认允许远程加载ObjectFactory变成了不允许。  
- RMI是在JDK 6u132, 7u122, 8u113版本开始做了限制。
- LDAP是JDK 11.0.1, 8u191, 7u201, 6u211版本开始做了限制。

### 2. JEP290防护机制 (Filter Incoming Serialization Data)
适用范围：  
JEP 290 在 JDK 9 中加入，但在 JDK 6,7,8 一些高版本中也添加了,JDK 8u121, 7u131, 6u141。
内容： 
防护机制规范原文：[JEP 290: Filter Incoming Serialization Data](https://openjdk.org/jeps/290)

- 提供一个限制反序列化类的机制，白名单或者黑名单
- 限制反序列化的深度和复杂度
- 为RMI远程调用对象提供了一个验证类的机制
- 定义一个可配置的过滤机制，比如可以通过配置properties文件的形式来定义过滤器

## 0x01 漏洞利用
不让远程加载类，那就需要利用本地类反序列化或者调用工厂类执行执行方法来到到目的。
- 找到一个受害者本地CLASSPATH中的类作为恶意的Reference Factory工厂类，并利用这个本地的Factory类执行命令。
公开常用的利用方法是通过Tomcat的`org.apache.naming.factory.BeanFactory` 工厂类去调用 `javax.el.ELProcessor#eval`方法或`groovy.lang.GroovyShell#evaluate`方法
- 利用LDAP直接返回一个恶意的序列化对象，JNDI注入依然会对该对象进行反序列化操作，利用反序列化Gadget完成命令执行。
通过LDAP的 `javaSerializedData`反序列化gadget

详细利用方法见：[探索高版本 JDK 下 JNDI 漏洞的利用方法](https://tttang.com/archive/1405/)

## Ref
- [探索高版本 JDK 下 JNDI 漏洞的利用方法](https://tttang.com/archive/1405/)
- [JNDI：JNDI-LDAP 注入及高版本JDK限制](https://m0d9.me/2020/07/23/JNDI-LDAP%20%E6%B3%A8%E5%85%A5%E5%8F%8A%E9%AB%98%E7%89%88%E6%9C%ACJDK%E9%99%90%E5%88%B6%E2%80%94%E2%80%94%E4%B8%8A/)
-[如何绕过高版本 JDK 的限制进行 JNDI 注入利用](https://paper.seebug.org/942/)
- [jdk21下的jndi注入](https://xz.aliyun.com/t/15265?time__1311=GqjxnD0D2AGQqGNeWxUxQTTxfx%3D3%3DkeW4D)
- [漫谈 JEP 290](https://xz.aliyun.com/t/10170)