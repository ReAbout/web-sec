# PHP 调试环境搭建
## 0x01 Vxcode远程调试
### （1）环境
Vscode + Ubuntu16.04虚拟机 + Xdebug

### （2） Install 

#### ubuntu 
- 首先安装好 Apache,Mysql,php   
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
#### VSCode
- 安装 php debug 插件   
- 创建默认php调试配置   

### （3） 使用
虚拟机开放ssh   
通过 ssh target方式连接虚拟机，就可以进行远程调试了。   

## 0x02 PhpStorm远程调试

- [可能是全网最详细的PhpStorm+xdebug远程调试php代码的教程](https://www.xiebruce.top/1191.html)

## 0x03 xdebug
> xdebug 2.X 和 xdebug 3.X 的php.ini配置文件不同

### xdebug 2.X 配置
```
[xdebug] 
zend_extension="/usr/local/lib/xdebug.so" 
xdebug.remote_autostart=1 
xdebug.remote_enable=1 
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
xdebug.client_host="192.168.117.1" 
xdebug.idekey="PHPSTORM" 
xdebug.remote_handler="dbgp"
```

