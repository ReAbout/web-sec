# PHP 调试环境搭建
## 0x01环境
Vscode + Ubuntu16.04虚拟机 + Xdebug

## 0x02 Install 

### ubuntu 
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
### VSCode
- 安装 php debug 插件   
- 创建默认php调试配置   

## 0x03 使用
虚拟机开放ssh   
通过 ssh target方式连接虚拟机，就可以进行远程调试了。   
