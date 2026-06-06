# Java 反序列化漏洞利用

## 一句话理解
Java 反序列化漏洞的核心，是应用把不可信字节流交给反序列化接口处理，攻击者再借助类库中的 Gadget 链，在对象恢复过程中触发方法调用、反射或命令执行。

## 基础理解
Java 序列化是把对象转换成字节序列，常见接口是 `ObjectOutputStream.writeObject()`。  
Java 反序列化是把字节序列恢复成对象，常见接口是 `ObjectInputStream.readObject()`。

对象要能被序列化，通常需要：
1. 实现 `java.io.Serializable`
2. 相关属性本身也可序列化，或被声明为瞬态字段

## 成立条件
典型需要同时满足：
1. 存在可控的反序列化入口
2. 运行环境中存在可利用的 Gadget 链

## 常见危害
- 远程代码执行
- 任意方法调用
- SSRF / JNDI 利用
- 文件写入
- 与框架组件联动形成更深利用链

## 常见入口点
### 1. 原生 Java 反序列化

```text
ObjectInputStream.readObject
ObjectInputStream.readUnshared
```

### 2. XML / YAML / JSON 等“非原生序列化”入口
这些问题不一定都属于原生 Java 序列化，但在实战里经常一起排查：

```text
XMLDecoder.readObject
Yaml.load
XStream.fromXML
ObjectMapper.readValue
JSON.parseObject
```

## 常见成因
- 接收来自网络、消息队列、缓存、文件的对象流并直接反序列化
- 依赖包中存在高危 Gadget
- 应用把 JNDI、RMI、XML、JSON 等入口错误地串进反序列化利用链

## 基础概念补充
### 1. JNDI
JNDI（Java Naming and Directory Interface）用于访问命名和目录服务，支持：
- DNS
- LDAP
- RMI
- CORBA

### 2. RMI
RMI（Remote Method Invocation）是 Java 远程方法调用机制，常见实现包括：
- JRMP
- CORBA 相关对象服务

这些机制经常与反序列化、JNDI 注入、远程类加载链条互相关联。

## 如何发现
### 1. 代码审计
优先搜索：
- `readObject`
- `readUnshared`
- `XMLDecoder`
- `XStream`
- `Yaml.load`
- `ObjectMapper.readValue`
- `JSON.parseObject`

### 2. 组件识别
重点识别是否存在高风险依赖：
- Commons Collections
- Spring
- Shiro
- XStream
- Fastjson
- 各类中间件和 RPC 框架

### 3. 输入源确认
关注：
- HTTP Body
- Cookie
- Session
- MQ 消息
- 缓存
- 文件上传内容

## 常见利用方向
### 1. Gadget 链利用
Java 反序列化的核心不是“构造一个危险对象就行”，而是利用类库中现成的调用链，把反序列化过程一路带到危险方法。

### 2. 框架与组件结合
以下场景常见于真实漏洞：

#### Spring
- Spring Framework 4.2.4 相关利用链
- POC: [spring-jndi](https://github.com/zerothoughts/spring-jndi)
- 分析：http://llfam.cn/2019/11/11/spring_4.2.4_unser/

#### Apache Commons Collections
- 这是 Java 反序列化史上最经典的 Gadget 来源之一
- 分析：https://www.iswin.org/2015/11/13/Apache-CommonsCollections-Deserialized-Vulnerability/

#### Fastjson
- 严格来说更常归类为反序列化 / AutoType / 反射链利用，但实战中经常与 Java 反序列化一起梳理
- 分析：https://paper.seebug.org/1318/

#### Shiro
- 常与 RememberMe、反序列化链、密钥问题联动
- 分析：https://paper.seebug.org/1503/

#### Apache Solr
- [Apache Solr 反序列化远程代码执行漏洞分析（CVE-2019-0192）](https://www.anquanke.com/post/id/210866?luicode=10000011&lfid=1076033957583411&featurecode=newtitle%0AUltrasonic+Fingerprint+ID+on+the+Galaxy+S10%3A+Pesto+Fingers&u=https%3A%2F%2Fwww.anquanke.com%2Fpost%2Fid%2F210866)

#### WebLogic
- [WebLogic CVE-2021-2394 反序列化漏洞分析](https://www.anquanke.com/post/id/249654)

## 实战排查思路
### 1. 先找入口，不先找 payload
入口点永远比 payload 更重要。

### 2. 再看依赖
确认类路径里到底有哪些可用 Gadget。

### 3. 再判断 JDK 和组件版本
同样的链，在不同 JDK 和组件版本下可用性会差很多。

### 4. 再决定利用路径
常见路径：
- 原生反序列化链
- JNDI / RMI / LDAP 结合
- 中间件自带 Gadget
- 通过第三方依赖链下沉到命令执行

## 防御要点
### 1. 不反序列化不可信数据
这是根本原则。

### 2. 优先使用安全数据格式
如 JSON，并避免将其恢复成可执行对象图。

### 3. 升级依赖与 JDK
很多反序列化风险都和老版本依赖和老 JDK 的默认行为有关。

### 4. 加入反序列化过滤
使用白名单、类型过滤、深度限制等机制限制可反序列化类型。

### 5. 最小化依赖面
类路径里不需要的高危依赖越少，可用 Gadget 越少。

## 速查清单
- 先搜反序列化入口，再识别依赖和版本
- 关注 `readObject`、`XStream`、`XMLDecoder`、`Yaml.load`
- 区分原生 Java 反序列化与 JSON/XML 类对象恢复问题
- 把 Spring、Commons Collections、Shiro、WebLogic、Solr 作为高频关注对象
- 结合 JNDI、RMI、LDAP、JDK 版本判断真实可利用性

## Reference
- [深入理解 JAVA 反序列化漏洞](https://paper.seebug.org/312/#2)
- https://yinwc.github.io/2020/02/08/java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E
- http://llfam.cn/2019/11/11/spring_4.2.4_unser/
- https://www.iswin.org/2015/11/13/Apache-CommonsCollections-Deserialized-Vulnerability/
- https://paper.seebug.org/1318/
- [Apache Solr反序列化远程代码执行漏洞分析（CVE-2019-0192）](https://www.anquanke.com/post/id/210866?luicode=10000011&lfid=1076033957583411&featurecode=newtitle%0AUltrasonic+Fingerprint+ID+on+the+Galaxy+S10%3A+Pesto+Fingers&u=https%3A%2F%2Fwww.anquanke.com%2Fpost%2Fid%2F210866)
- [WebLogic CVE-2021-2394 反序列化漏洞分析](https://www.anquanke.com/post/id/249654)
