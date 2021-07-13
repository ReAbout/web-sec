# 反序列漏洞-PHP

## 0x01 Introduction

### 反序列化：
直接看例子   
```
<?php 
    class test
    {
        private $flag = "flag{233}";
        public $a = "aaa";
        static $b = "bbb";
    }

    $test = new test;
    $data = serialize($test);
    echo $data;
 ?>
```
序列化的结果   
```
O:4:"test":2:{s:10:"testflag";s:9:"flag{233}";s:1:"a";s:3:"aaa";}
O:<class_name_length>:"<class_name>":<number_of_properties>:{<properties>}
```

### 漏洞利用思路：
跟二进制漏洞利用很相似，寻找gadgat构造POP链(Property-Oriented Programing)，实现漏洞利用。    
1. 查找魔术函数寻找可以理由的逻辑代码。
2. 利用倒叙思想一步步寻找相关的同名函数或变量。
3. 最后构造成触发的POP链。



## 0x02 魔术函数
### 可利用的点

* 反序列化会调用   __wakeup()，__destruct()
* 存在不可访问方法 __call(),__callStatic()
* 对象打印  __toString()

### 常用魔术函数
* __construct()，类的构造函数 
* __destruct()，类的析构函数 
* __call()，在对象中调用一个不可访问方法时调用 
* __callStatic()，用静态方式中调用一个不可访问方法时调用 
* __get()，获得一个类的成员变量时调用 
* __set()，设置一个类的成员变量时调用 
* __isset()，当对不可访问属性调用isset()或empty()时调用 
* __unset()，当对不可访问属性调用unset()时被调用。 
* __sleep()，执行serialize()时，先会调用这个函数 
* __wakeup()，执行unserialize()时，先会调用这个函数 
* __toString()，类被当成字符串时的回应方法 
* __invoke()，调用函数的方式调用一个对象时的回应方法 
* __set_state()，调用var_export()导出类时，此静态方法会被调用。
* __clone()，当对象复制完成时调用 
* __autoload()，尝试加载未定义的类 
* __debugInfo()，打印所需调试信息

## 0x03 触发反序列化

### unserialize()

### phar反序列化
常用攻击链 = 上传文件 + PoP链 + 文件操作phar://伪协议   
- [利用 phar 拓展 php 反序列化漏洞攻击面](https://paper.seebug.org/680/)   



## 0x04 bypass技巧

### 跳过__wakeup 执行


这里就要用到CVE-2016-7124漏洞，当序列化字符串中表示对象属性个数的值大于真实的属性个数时会跳过__wakeup的执行
```
构造序列化对象：O:5:"SoFun":1:{S:7:"\00*\00file";s:8:"flag.php";}
绕过__wakeup：O:5:"SoFun":2:{S:7:"\00*\00file";s:8:"flag.php";}
```

### 字符溢出控制序列化内容

序列化字符串中部分数据可控，可以填充字符达到序列化当前目标上限，伪造后面数据。   
![](https://img2018.cnblogs.com/blog/1077935/201910/1077935-20191017112220979-1117655826.png)
[从一道ctf看php反序列化漏洞的应用场景](https://www.cnblogs.com/litlife/p/11690918.html)

### `O:+8` 绕过正则检测`/O:\d:/`


[红日安全代码审计Day11 - unserialize反序列化漏洞](https://xz.aliyun.com/t/2733)

## 0x05 技巧篇


### 利用array()动态调用类

```
<?php
class A{
    public function test(){
        echo "aaaa";
    }
}
$tr = array(new A(),"test");
$tr();//这样就可以直接调用到A的test函数
```

### 利用array()加载类
unserialize后进入autoload加载类。   

[国赛2020 WriteUp](https://blog.csdn.net/qq_42697109/article/details/108212765)



## Reference
入门篇
* [PHP反序列化由浅入深](https://xz.aliyun.com/t/3674)
* [php 反序列化POP链的构造与理解](https://blog.szfszf.top/tech/php-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96pop%E9%93%BE%E7%9A%84%E6%9E%84%E9%80%A0%E4%B8%8E%E7%90%86%E8%A7%A3/)
