# 反序列漏洞-PHP

## 一句话理解
PHP 反序列化漏洞的本质，是应用把用户可控数据传给 `unserialize()` 后，攻击者借助目标类中的魔术方法、危险调用点或已有 POP 链，在对象恢复和销毁过程中触发敏感操作。

## 成立条件
通常需要同时具备以下条件：
1. 存在可控的反序列化入口，例如 `unserialize($_GET['data'])`
2. 目标环境中存在可利用的类、魔术方法或可拼接的 POP 链

## 漏洞成因
典型示例：

```php
class VulnerableClass {
    private $data;

    public function __destruct() {
        system($this->data);
    }
}

$obj = unserialize($_GET['data']);
```

如果攻击者能控制 `$data` 对象属性，就可能在析构阶段触发命令执行。

## 序列化基础
### 1. 序列化格式
示例：

```php
<?php
class test
{
    private $flag = "flag{233}";
    public $a = "aaa";
    static $b = "bbb";
}

$test = new test;
$data = serialize($test);
echo $data;
?>
```

序列化结果示意：

```text
O:4:"test":2:{s:10:"testflag";s:9:"flag{233}";s:1:"a";s:3:"aaa";}
O:<class_name_length>:"<class_name>":<number_of_properties>:{<properties>}
```

### 2. 需要关注的点
- 类名
- 属性数量
- 属性可见性
- 属性名长度
- 值类型与长度

> 私有属性和受保护属性在序列化结果中会带上特殊前缀，构造 payload 时需要特别注意。

## 常见利用目标
- 文件删除、文件读取、文件写入
- 命令执行
- 任意函数调用
- SSRF
- 包含链利用
- 与 `phar://`、文件上传、LFI 联动

## 利用思路
可以把 PHP 反序列化理解为“从可控对象出发，沿着已有类的方法调用关系拼接出危险执行路径”。

典型步骤：
1. 找到可控反序列化入口
2. 枚举已加载类和魔术方法
3. 找危险函数点，如 `system`、`eval`、`include`、文件操作、回调调用
4. 倒推需要满足的属性和值
5. 构造触发链并验证执行时机

这类链通常称为 POP 链（Property-Oriented Programming）。

## 常见魔术方法
### 1. 高价值触发点
- `__wakeup()`
- `__destruct()`
- `__toString()`
- `__invoke()`
- `__call()`
- `__callStatic()`
- `__get()`
- `__set()`

### 2. 常见魔术方法列表
- `__construct()`：构造函数
- `__destruct()`：析构函数
- `__call()`：调用不可访问方法时触发
- `__callStatic()`：静态调用不可访问方法时触发
- `__get()`：读取不可访问属性时触发
- `__set()`：写入不可访问属性时触发
- `__isset()`：对不可访问属性做 `isset/empty` 时触发
- `__unset()`：对不可访问属性做 `unset` 时触发
- `__sleep()`：执行 `serialize()` 时触发
- `__wakeup()`：执行 `unserialize()` 时触发
- `__toString()`：对象转字符串时触发
- `__invoke()`：对象当函数调用时触发
- `__set_state()`：`var_export()` 导出时触发
- `__clone()`：对象克隆时触发
- `__autoload()`：尝试加载未定义类
- `__debugInfo()`：调试输出时触发

## 常见入口点
### 1. `unserialize()`
最直接的入口，也是最常见的审计点。

### 2. `phar://` 反序列化
很多文件操作函数在处理 `phar://` 路径时，会解析 Phar 元数据，从而触发反序列化。

典型组合：
- 文件上传
- 构造 Phar
- 通过文件操作函数触发
- 配合 POP 链利用

- [利用 phar 拓展 php 反序列化漏洞攻击面](https://paper.seebug.org/680/)

## 常见审计点
### 1. 危险函数
重点关注链末端是否能触达：
- `system`
- `exec`
- `passthru`
- `shell_exec`
- `eval`
- `assert`
- `include` / `require`
- 文件读写删除函数
- 回调函数，如 `call_user_func`

### 2. 触发时机
并不是只有 `__wakeup()` 能触发，很多链真正生效是在：
- 脚本结束时的 `__destruct()`
- 字符串拼接时的 `__toString()`
- 间接回调中的 `__invoke()`

### 3. 自动加载
如果类不存在但存在自动加载器，也可能扩大可利用面。

## 常见绕过技巧
### 1. 跳过 `__wakeup()`
- [php 反序列化中 wakeup 绕过总结](https://fushuling.com/index.php/2023/03/11/php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%ADwakeup%E7%BB%95%E8%BF%87%E6%80%BB%E7%BB%93/)

#### CVE-2016-7124
当序列化字符串中声明的属性个数大于真实属性个数时，某些版本中可跳过 `__wakeup()`：

```text
构造序列化对象：O:5:"SoFun":1:{S:7:"\00*\00file";s:8:"flag.php";}
绕过 __wakeup：O:5:"SoFun":2:{S:7:"\00*\00file";s:8:"flag.php";}
```

### 2. 引用赋值与对象关系利用
PHP 序列化支持引用，很多题目和真实场景会利用引用关系改变触发顺序或对象状态。

### 3. Fast-destruct
通过对象嵌套、异常流程或引用处理，让析构更早触发，从而绕过某些限制。

### 4. 序列化字符串拼接与字符逃逸
当序列化内容的一部分可控时，可能通过长度错位、字符填充影响后续结构。

相关资料：
- [从一道 CTF 看 PHP 反序列化漏洞的应用场景](https://www.cnblogs.com/litlife/p/11690918.html)

![](https://img2018.cnblogs.com/blog/1077935/201910/1077935-20191017112220979-1117655826.png)

### 5. 绕过正则检测
有些业务只用类似 `/O:\d:/` 的弱正则判断对象序列化格式，这类限制经常可以被变形绕过，例如：
- `O:+8`
- 大小写与编码差异
- 其他类型包装后再还原

- [红日安全代码审计 Day11 - unserialize 反序列化漏洞](https://xz.aliyun.com/t/2733)

## 实战技巧
### 1. 动态调用
有些链条会利用数组回调、可调用对象等机制：

```php
<?php
class A {
    public function test() {
        echo "aaaa";
    }
}
$tr = array(new A(), "test");
$tr();
```

### 2. 自动加载类
某些情况下，`unserialize()` 后会进入自动加载逻辑，从而扩大可利用类范围。

- [国赛 2020 WriteUp](https://blog.csdn.net/qq_42697109/article/details/108212765)

## 实战排查思路
### 1. 先找入口
高频点：
- Cookie
- Session
- GET / POST 参数
- 缓存字段
- 队列消息
- 文件内容
- Phar 文件路径

### 2. 再找可利用类
重点看：
- 框架类
- 依赖包类
- 文件系统相关类
- 日志、模板、数据库、缓存组件

### 3. 最后拼链
从危险函数往回逆推，比从入口正向乱试更高效。

## 防御要点
### 1. 不要对不可信数据使用 `unserialize()`
这是最根本的防御。

### 2. 优先使用安全数据格式
例如 JSON，只反序列化纯数据，不恢复对象行为。

### 3. 启用白名单
如果必须反序列化对象，使用允许类白名单，限制可实例化对象范围。

### 4. 减少危险魔术方法
避免在 `__destruct()`、`__wakeup()`、`__toString()` 中做高风险操作。

### 5. 谨慎处理 `phar://`
上传、文件操作、包含链应评估 Phar 元数据带来的反序列化面。

## 速查清单
- 先找 `unserialize()` 和 `phar://` 入口
- 先枚举魔术方法，再找危险函数和可控属性
- 关注 `__destruct()`、`__toString()`、`__invoke()` 等延迟触发点
- 检查是否存在类自动加载、依赖包 gadget、文件操作链
- 检查是否能与上传、包含、日志、Session、Phar 组合利用

## Reference
- [PHP 反序列化由浅入深](https://xz.aliyun.com/t/3674)
- [php 反序列化 POP 链的构造与理解](https://blog.szfszf.top/tech/php-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96pop%E9%93%BE%E7%9A%84%E6%9E%84%E9%80%A0%E4%B8%8E%E7%90%86%E8%A7%A3/)
