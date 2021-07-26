# 错综复杂的后端逻辑及安全

## 0x00 前言
Web安全不同于二进制安全，更多在于经验的积累和知识体系的广度，应为在上层应用架构中，让web看似入门简单，因为复杂的架构和技术栈，其实精通很有难度。而并非像二进制基础知识具有很好的通用性和拓展性。        

## 0x01 分层（纵向）

#---用户侧---#
- 前端（html，JavaScript等）
- Web应用（CMS，OA，ERP等）
- MVC框架&组件（Thinkphp，FCKeditor等）
- 运行语言（PHP，.NET，Java Web等）
- Web中间件（Apache，Nginx，Tomcat等）
- 数据库系统（Mysql，Mssql等）
- 操作系统（Linux，Windows等）
#---硬件侧---#

## 0x02 架构（横向）
通过[千万级并发下，淘宝服务端架构如何演进？](https://developer.51cto.com/art/201906/597895.htm)就可以知道，对于用户侧呈现的web页面，其实背后可能是单个服务器支撑，也可能是复杂的架构系统。    

### 单机架构
对于攻击者而言，web服务和数据库系统都处于一台服务器中，结构简单。    
不同攻击面漏洞可以相互影响，例如：数据库写操作可以写入web执行路径中进而getshell。     
![](https://s2.51cto.com/oss/201906/14/a02337d35e38630cb4e0c44b88e8b983.jpg-wh_651x-s_3331278464.jpg)

### 以云平台承载系统
对于攻击者而言，系统复杂了，攻击面也细化成不同方向。      
![](https://s3.51cto.com/oss/201906/14/97b88fa7fb4f64aecd4701b12bef38b6.jpg-wh_600x-s_2360315611.jpg)

## 0x03 分类（漏洞）



## Ref

- https://developer.51cto.com/art/201906/597895.htm
