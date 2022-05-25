# Javascript 原型链污染

###  prototype & `__proto__`
* prototype是一个类的属性，所有类对象在实例化的时候将会拥有prototype中的属性和方法
* 一个对象的__proto__属性，指向这个对象所在的类的prototype属性

所有类对象在实例化的时候将会拥有prototype中的属性和方法，这个特性被用来实现JavaScript中的继承机制。

 这篇文章[CVE-2019-11358分析](https://xz.aliyun.com/t/11272)分析对JavaScript原型链继承机制分析的很透彻。


## Ref
- [CVE-2019-11358分析](https://xz.aliyun.com/t/11272)
- [深入理解 JavaScript Prototype 污染攻击](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)
