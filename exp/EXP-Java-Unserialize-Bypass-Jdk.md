# 绕过过高版本Jdk的限制进行Jndi注入利用

## 一句话理解
高版本 JDK 对经典 JNDI 注入链做了多轮收紧，但这并不等于 JNDI 完全不可利用。核心变化是“远程类加载”越来越难，而利用思路逐渐转向“本地类利用”“本地工厂利用”和“反序列化链利用”。

## 为什么高版本变难了
### 1. `trustURLCodebase = false`
历史上，攻击者常通过 RMI / LDAP 返回恶意 `Reference`，诱导目标从远程加载恶意类。

后来高版本 JDK 开始限制这类行为：
- RMI 从 `JDK 6u132`、`7u122`、`8u113` 起收紧
- LDAP 从 `JDK 11.0.1`、`8u191`、`7u201`、`6u211` 起收紧

也就是远程 `ObjectFactory` 加载不再默认允许。

### 2. JEP 290 反序列化过滤
JEP 290 引入了输入过滤机制，用来限制反序列化类、深度和复杂度。

适用范围：
- JDK 9 正式引入
- JDK 8u121、7u131、6u141 等高版本也补入了类似能力

规范原文：
- [JEP 290: Filter Incoming Serialization Data](https://openjdk.org/jeps/290)

核心能力：
- 限制允许反序列化的类
- 限制对象深度和复杂度
- 为 RMI 调用提供类过滤机制
- 支持通过配置定义过滤策略

## 高版本下的利用思路
经典“远程类加载”受限后，常见思路转为：
- 利用受害者本地 CLASSPATH 中已有类
- 利用本地工厂类触发方法调用
- 通过 LDAP 返回序列化对象，走反序列化 Gadget 链

## 常见利用方向
### 1. 本地 `Reference Factory` 利用
思路是：
- 不再依赖远程恶意类
- 改为找目标本地已有的可利用工厂类
- 让 JNDI 返回的 `Reference` 调用这些本地类完成危险行为

公开高频思路是利用 Tomcat 的：
- `org.apache.naming.factory.BeanFactory`

再进一步调用：
- `javax.el.ELProcessor#eval`
- `groovy.lang.GroovyShell#evaluate`

### 2. LDAP 返回序列化对象
另一条常见思路是：
- LDAP 不返回远程类加载对象
- 而是直接返回恶意序列化数据
- 目标在处理 `javaSerializedData` 时触发本地 Gadget 链

也就是把 JNDI 注入与 Java 反序列化利用结合起来。

## 与普通 JNDI 注入的区别
高版本场景下，重点不再是“起一个恶意类服务器就能打”，而是：
- 目标本地有哪些类
- 应用是否带 Tomcat、Groovy、EL 等组件
- 是否存在可用 Gadget
- JEP 290 是否放行相关类

## 实战排查思路
### 1. 先看 JDK 版本
这是判断是否还能走经典链的第一步。

### 2. 再看目标类路径
重点确认是否存在：
- Tomcat 相关类
- EL 相关类
- Groovy
- 常见反序列化 Gadget 依赖

### 3. 再看协议与返回对象类型
区分：
- RMI 路径
- LDAP 路径
- `Reference` 利用
- `javaSerializedData` 利用

### 4. 再看 JEP 290 过滤
即使有 Gadget，也可能被过滤器挡掉。

## 防御要点
### 1. 升级 JDK 不是终点
高版本只是缩小经典利用面，不代表所有 JNDI 风险都消失。

### 2. 禁止不可信 JNDI 输入
不要把用户输入直接交给 `lookup()`。

### 3. 限制可访问协议与命名源
明确限制 LDAP、RMI 等外部命名服务的访问。

### 4. 使用反序列化过滤
结合 JEP 290 或应用层白名单进一步减少可利用面。

### 5. 最小化依赖
目标类路径里的 Tomcat、Groovy、EL、Commons 类越少，可利用面越小。

## 速查清单
- 先确认 JDK 版本和 `trustURLCodebase` 限制范围
- 再确认目标本地是否存在 BeanFactory、EL、Groovy 等可利用类
- 再判断是 `Reference` 利用还是 `javaSerializedData` 反序列化利用
- 检查是否启用了 JEP 290 以及过滤策略
- 不把“高版本 JDK”误判为“天然安全”

## Reference
- [探索高版本 JDK 下 JNDI 漏洞的利用方法](https://tttang.com/archive/1405/)
- [JNDI：JNDI-LDAP 注入及高版本JDK限制](https://m0d9.me/2020/07/23/JNDI-LDAP%20%E6%B3%A8%E5%85%A5%E5%8F%8A%E9%AB%98%E7%89%88%E6%9C%ACJDK%E9%99%90%E5%88%B6%E2%80%94%E2%80%94%E4%B8%8A/)
- [如何绕过高版本 JDK 的限制进行 JNDI 注入利用](https://paper.seebug.org/942/)
- [jdk21下的 jndi 注入](https://xz.aliyun.com/t/15265?time__1311=GqjxnD0D2AGQqGNeWxUxQTTxfx%3D3%3DkeW4D)
- [漫谈 JEP 290](https://xz.aliyun.com/t/10170)
