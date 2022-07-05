# Java 调试环境搭建

## 0x00 准备

- IDEA

## 0x01 远程调试

### 1. Tomcat 远程调试

**远程Tomcat配置**： 在catalina.sh 中添加 address 调试端口 9090

```
CATALINA_OPTS="-server -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=9090" 

```

**IDEA配置**：
https://blog.51cto.com/u_15127502/3518074
