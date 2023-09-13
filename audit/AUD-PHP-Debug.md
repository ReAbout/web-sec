# PHP 调试环境搭建
## 0x01 服务侧安装配置
### Linux远程调试
#### （1）环境
Ubuntu16.04虚拟机 + Xdebug
#### （2） Install on ubuntu
- 首先安装好 Apache,Mysql,php   -
```
sudo apt install apache2 php  libapache2-mod-php mysql-server
```
- xdebug install
```
sudo apt-get install php-xdebug
```
- 可以查看是否安装成功：   
```
php -m
```
- `/etc/php/7.0/apache2/php.ini`添加如下配置：   
```
xdebug.remote_enable = 1
xdebug.remote_autostart = 1
```
- 一切配置好了，记得好重启apache2 服务。

#### （3） 使用
虚拟机开放ssh   
通过 ssh target方式连接虚拟机，就可以进行远程调试了。   

### Windows远程调试
### （1）环境
Windows系统 + Xdebug
### （2）Install on windows
可以直接找对应的php版本的xdebug，都是编译好的。    
https://xdebug.org/download/historical
配置php.ini，详情见xdebug配置
## 0x02 客户端工具配置

#### VSCode
- 安装 php debug 插件   
- 创建默认php调试配置   

#### phpstrom
- [可能是全网最详细的PhpStorm+xdebug远程调试php代码的教程](https://www.xiebruce.top/1191.html)

## 0x03 xdebug配置
> xdebug 2.X 和 xdebug 3.X 的php.ini配置文件不同

### xdebug 2.X 配置
```
[xdebug] 
zend_extension="/usr/local/lib/xdebug.so" 
xdebug.remote_autostart=1 
xdebug.remote_enable=1
;客户端ip
xdebug.remote_host = "192.168.117.1" 
xdebug.idekey="PHPSTORM" 
xdebug.remote_handler=dbgp 
xdebug.remote_port=9000
```
### xdebug 3.X 配置
```
[xdebug] 
zend_extension="/usr/local/lib/xdebug.so" 
xdebug.mode=debug 
xdebug.discover_client_host=true 
xdebug.client_port=9000
;客户端ip
xdebug.client_host="192.168.117.1" 
xdebug.idekey="PHPSTORM" 
xdebug.remote_handler="dbgp"
```


## 0x04 php报错开启

### php单文件： 
```
ini_set("display_errors", "On");
error_reporting(E_ALL | E_STRICT)
```
### 修改php.ini：
```
display_errors = On
error_reporting  =  E_ALL & ~E_NOTICE

```

