# 命令注入&代码执行-Java

## 如何发现？

Java中命令执行的类/函数
```
java.lang.Runtime
java.lang.ProcessBuilder
java.lang.ProcessImpl
javax.script.ScriptEngineManager
groovy.lang.GroovyShell
```

### java.lang.Runtime
`Runtime.getRuntime().exec(cmd)` 
非/bin/sh和cmd的程序命令执行，无法命令注入，可以考虑参数注入。   
参考：[GTFOBins](https://gtfobins.github.io/)，总结Linux命令中可执行命令、文件读写等操作的参数。
### java.lang.ProcessBuilder
```
StringBuilder sb = new StringBuilder();

  try {
      //String[] arrCmd = {"/bin/sh", "-c", cmd}; //linux
      String[] arrCmd = {cmd};                  //windows,windos下无需指定
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

### java.lang.ProcessImpl
```
Class clazz = Class.forName("java.lang.ProcessImpl");
Method start = clazz.getDeclaredMethod("start", String[].class, Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);
start.setAccessible(true);
start.invoke(null, new String[]{"calc"}, null, null, null, false);
```
### javax.script.ScriptEngineManager
```
/*jsurl
var a = mainOutput(); 
function mainOutput() { 
  var x=java.lang.Runtime.getRuntime().exec("calc");
}
*/
ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");//通过文件扩展名获取：
//ScriptEngine engine = manager.getEngineByName("JavaScript");//通过脚本名称获取：     
//ScriptEngine engine = manager.getEngineByMimeType("text/javascript");  //通过MIME类型来获取： 
Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);//启动javascript引擎
String cmd = String.format("load(\"%s\")", jsurl);//可直接远程加载jsurl=http://127.0.0.1/re.js
engine.eval(cmd, bindings);
```
### groovy.lang.GroovyShell
```
public void groovyshell(String content) {
    GroovyShell groovyShell = new GroovyShell();
    groovyShell.evaluate(content);
}
```

## Ref
- [Java代码审计之RCE（远程命令执行）](https://blog.51cto.com/u_13963323/5066457)
