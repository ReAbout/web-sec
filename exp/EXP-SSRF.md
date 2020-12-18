# 后端安全-代码逻辑-SSRF
## 0x01 Introduction
SSRF(Server-Side Request Forgery:服务器端请求伪造) 
## 0x02 漏洞发现
### 漏洞触发点
1. 社交分享功能：获取超链接的标题等内容进行显示
2. 转码服务：通过URL地址把原地址的网页内容调优使其适合手机屏幕浏览
3. 在线翻译：给网址翻译对应网页的内容
4. 图片加载/下载：例如富文本编辑器中的点击下载图片到本地；通过URL地址加载或下载图片
5. 图片/文章收藏功能：主要其会取URL地址中title以及文本的内容作为显示以求一个好的用具体验
6. 云服务厂商：它会远程执行一些命令来判断网站是否存活等，所以如果可以捕获相应的信息，就可以进行ssrf测试
7. 网站采集，网站抓取的地方：一些网站会针对你输入的url进行一些信息采集工作
8. 数据库内置功能：数据库的比如mongodb的copyDatabase函数
9. 邮件系统：比如接收邮件服务器地址
10. 编码处理, 属性信息处理，文件处理：比如ffpmg，ImageMagick，docx，pdf，xml处理器等
11. 未公开的api实现以及其他扩展调用URL的功能：可以利用google 语法加上这些关键字去寻找SSRF漏洞一些的url中的关键字：share、wap、url、link、src、source、target、u、3g、display、sourceURl、imageURL、domain……
12. 从远程服务器请求资源（upload from url 如discuz！；import & expost rss feed 如web blog；使用了xml引擎对象的地方 如wordpress xmlrpc.php）

Ref:[了解SSRF,这一篇就足够了](https://xz.aliyun.com/t/2115)
## 0x03 漏洞利用
主要用于内网渗透
作为代理，用于内网探测和攻击
### dict://
### gophar://
### file://
### http:// 结合 Redis 利用

## 0x04 Bypass
### Use DNS Rebinding to Bypass SSRF in Java
`http://test.com/checkssrf?url=http://dns_rebind.reabout.com` 
利用条件：
1. 可以修改NDS解析的域名
2. 业务逻辑通过IP判断是否内网进行过滤
3. TTL =0
[Use DNS Rebinding to Bypass SSRF in Java](https://paper.seebug.org/390/)

