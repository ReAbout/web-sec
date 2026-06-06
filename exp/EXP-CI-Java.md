# 命令注入&代码执行-Java

## 一句话理解
Java 场景下的命令执行，不一定都表现为“传统 shell 注入”，很多时候是开发直接调用 `Runtime`、`ProcessBuilder`、脚本引擎或表达式引擎，导致用户输入能够触发系统命令、脚本执行或代码执行。

## 常见危险类与接口

```text
java.lang.Runtime
java.lang.ProcessBuilder
java.lang.ProcessImpl
javax.script.ScriptEngineManager
groovy.lang.GroovyShell
```

## 常见危害
- 执行系统命令
- 文件读写
- 下载并执行恶意程序
- 反弹 Shell
- 结合表达式注入、反序列化、模板注入达成 RCE

## 实战理解
Java 命令执行常见分成两类：
1. 直接命令执行：开发显式调用系统命令
2. 间接代码执行：用户输入进入脚本引擎、表达式引擎、Groovy 等动态执行组件

## 典型危险点
### 1. `java.lang.Runtime`

```java
Runtime.getRuntime().exec(cmd)
```

重点：
- `exec()` 默认不是把整串命令交给 `/bin/sh -c` 或 `cmd /c`
- 因此很多场景并非经典“分隔符命令注入”，而更偏向“参数注入”
- 如果开发手动套了 shell，例如 `"/bin/sh", "-c", cmd"`，才更接近传统命令拼接注入

参考：
- [GTFOBins](https://gtfobins.github.io/)

### 2. `java.lang.ProcessBuilder`

```java
StringBuilder sb = new StringBuilder();

try {
    String[] arrCmd = {cmd};
    ProcessBuilder processBuilder = new ProcessBuilder(arrCmd);
    Process p = processBuilder.start();
    BufferedInputStream in = new BufferedInputStream(p.getInputStream());
    BufferedReader inBr = new BufferedReader(new InputStreamReader(in));
    String tmpStr;

    while ((tmpStr = inBr.readLine()) != null) {
        sb.append(tmpStr);
    }
} catch (Exception e) {
    return e.toString();
}

return sb.toString();
```

重点关注：
- 参数数组是如何构造的
- 是否把用户输入直接作为完整命令
- 是否显式调用 shell

### 3. `java.lang.ProcessImpl`

```java
Class clazz = Class.forName("java.lang.ProcessImpl");
Method start = clazz.getDeclaredMethod("start", String[].class, Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);
start.setAccessible(true);
start.invoke(null, new String[]{"calc"}, null, null, null, false);
```

这类写法常见于反射调用或绕过某些表层审计规则的场景。

### 4. `javax.script.ScriptEngineManager`

```java
/*jsurl
var a = mainOutput();
function mainOutput() {
  var x = java.lang.Runtime.getRuntime().exec("calc");
}
*/
ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");
Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);
String cmd = String.format("load(\"%s\")", jsurl);
engine.eval(cmd, bindings);
```

重点：
- 动态脚本执行本身就极其危险
- 若支持远程加载脚本，风险更高
- 需关注 JDK 版本、脚本引擎实现和禁用情况

### 5. `groovy.lang.GroovyShell`

```java
public void groovyshell(String content) {
    GroovyShell groovyShell = new GroovyShell();
    groovyShell.evaluate(content);
}
```

Groovy 一旦接收用户可控表达式，通常很容易演变为高危代码执行。

## 常见利用场景
- 后台执行诊断命令
- 插件系统、脚本系统、规则引擎
- 表达式求值功能
- 调试接口
- 任务调度、运维平台、CI/CD 平台
- 动态模板或 Groovy 扩展点

## 常见排查思路
### 1. 先找危险 API
代码审计重点搜：
- `Runtime.getRuntime().exec`
- `new ProcessBuilder`
- `ScriptEngineManager`
- `GroovyShell`
- 反射调用命令执行类

### 2. 再看输入如何进入
重点确认：
- 是作为完整命令
- 还是作为参数
- 还是进入脚本/表达式字符串

### 3. 判断是否经过 shell
这会直接影响利用方式：
- 若经过 shell，优先尝试经典命令注入思路
- 若未经过 shell，更多考虑参数注入和程序行为利用

### 4. 看是否有回显
区分：
- 直接回显标准输出
- 报错回显
- 无回显但可出网

## 常见绕过思路
### 1. 参数注入
Java 里很多所谓“命令注入”本质是向系统命令追加危险参数，而不是简单拼 `;`。

### 2. 平台差异
Windows 和 Linux 在：
- shell 语法
- 路径
- 可执行程序
- 参数格式

上都不同，payload 不能混用。

### 3. 间接执行链
即使找不到 `Runtime.exec()`，也要继续看：
- 脚本引擎
- Groovy
- 表达式引擎
- 反序列化链

## 防御要点
### 1. 避免直接调用系统命令
优先使用 Java 原生 API 处理文件、网络、压缩等操作。

### 2. 使用固定参数模板
如必须调用外部程序，命令和参数都应做严格白名单，不允许用户传完整命令行。

### 3. 禁止动态脚本执行
不要把用户输入传给 `ScriptEngine`、`GroovyShell` 或类似接口。

### 4. 最小权限
运行 Java 服务的账户不应具备高权限和敏感目录写权限。

## 速查清单
- 先搜 `Runtime`、`ProcessBuilder`、`ScriptEngineManager`、`GroovyShell`
- 先分清命令执行、参数注入还是脚本执行
- 判断是否经过 shell，再选择利用思路
- 检查是否存在标准输出回显、错误回显或外带能力
- 继续联动表达式注入、模板注入和反序列化链

## Reference
- [Java 代码审计之 RCE（远程命令执行）](https://blog.51cto.com/u_13963323/5066457)
