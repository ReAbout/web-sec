# Java 调试环境搭建

## 0x00 准备

- IDEA

## 0x01 远程调试

### 1. Tomcat 远程调试

#### 远程Tomcat配置：    

> jdk9+ 地址参数采用 address=*:5005
  
**方案一**：   
在./bin/catalina.sh 中添加 address 调试端口 5005
```
CATALINA_OPTS="-server -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005" //jdk5-8
```
或者是
```
JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,address=5005,suspend=n,server=y"
```
**方案二**：


修改./bin/startup.sh，在最后一行exec语句中添加 jpda

```
exec "$PRGDIR"/"$EXECUTABLE" jpda start "$@"
```
监听的端口可以在./bin/catalina.sh中修改

```
if [ -z "$JPDA_ADDRESS" ]; then
 JPDA_ADDRESS="0.0.0.0:5005"
```

重启服务即可

#### 调试的参数说明：
1. transport
指定运行的被调试应用和调试者之间的通信协议，它由几个可选值：
dt_socket：主要的方式，采用 socket 方式连接
dt_shmem：采用共享内存方式连接，仅支持 Windows 平台（暂未验证）
2. server
当前应用作为调试服务端还是客户端，默认为 n。
如果你想将当前应用作为被调试应用，设置该值为 y；如果你想将当前应用作为客户端，作为调试的发起者，设置该值为 n。
3. suspend
当前应用启动后，是否阻塞应用直到被连接，默认值为 y。
在大部分的应用场景，这个值为 n，即不需要应用阻塞等待连接。一个可能为 y 的应用场景是，你的程序在启动时出现了一个故障，为了调试，必须等到调试方连接上来后程序再启动。
4. address
暴露的调试连接端口，默认值为 8000。
5. onthrow
当程序抛出设定异常时，中断调试。
6. onuncaught
当程序抛出未捕获异常时，是否中断调试，默认值为 n。
7. launch
当调试中断时，执行的程序。
8. timeout
该参数限定为 java -agentlib:jdwp=… 可用，单位为毫秒ms。
当 suspend = y 时，该值表示等待连接的超时；当 suspend = n 时，该值表示连接后的使用超时。

### 2. Jar 远程调试
```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005  //jdk5-8
-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005  //jdk9+
```

## 0x02 客户端配置
### 1. IDEA配置
https://blog.51cto.com/u_15127502/3518074

## Ref
- https://cloud.tencent.com/developer/article/1917396 
- https://blog.51cto.com/u_15127502/3518074
