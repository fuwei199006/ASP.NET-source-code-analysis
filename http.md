#### 从打开一个网址说起

当在浏览器中输入一个网址的时候，浏览器会渲染出对应的网页的内容。作为web开发人员来说，应该知道这个过程：

1. 当输入的一个网址为域名的时候，浏览器则根据本机的网关和DNS服务器来解析出访问的真正的IP地址。如果是直接访问IP则直接与服务器通信，发送请求。 请求原理简单如下：

   ![DNS请求原理](/assets/DNS请求原理.png)  
   \(此处只是简单表示下域名解析原理，解析过程比这个复杂的多。\)

2. 发送请求的时候会经历TCP三次握手过程（http也是基于TCP的协议），当TCP连接建立成功后，浏览器会根据http协议，把请求的内容封装成请求报文，发送给web服务器.

1. 服务器会根据请求的报文的内容，执行对应的程序和读取对应的文件，按照http协议的规则返回响应内容\(包括header和body\)。    

1. 服务器根据响应头来解析响应的内容，完成html+css+js的渲染和执行。

#### http协议

> HTTP协议是Hyper Text Transfer Protocol（超文本传输协议）的缩写,是用于从万维网（WWW:World Wide Web ）服务器传输超文本到本地浏览器的传送协议。

用简单的话来说，当客户端与服务器端通信的时候，需要传输的内容有（HTML 文件，js+css, 图片，文本, 查询结果等）,http协议把内容传输规范化。可以随便查看下一个http协议的内容：

 ![请求jquery](/assets/jquery请求.png)       
 
 （上面的图是请求jqury的请求和响应信息）
 
 详细看下请求的头信息      
 
 ``` json
 
:authority:cdn.bootcss.com
:method:GET
:path:/jquery/2.0.3/jquery.min.js
:scheme:https
accept:*/*
accept-encoding:gzip, deflate, sdch, br
accept-language:zh-CN,zh;q=0.8,en;q=0.6
cache-control:no-cache
pragma:no-cache
referer:http://www.runoob.com/http/http-intro.html
user-agent:Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36    

 ```    
 
 上面有几个重点的信息:    
 
 - **method:**  method代表请求的方式（常用的有Get,Post,Put,Delete）    
 
 - **accept-encoding:**  是浏览器发给服务器,声明浏览器支持的编码类型    
 
 - **referer:** 告诉服务器我是从哪个页面链接过来的     
 
 - **user-agent:** 当前设备的一些信息
 

**HTTP三点注意事项：**

* HTTP是无连接：无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。

* HTTP是媒体独立的：这意味着，只要客户端和服务器知道如何处理的数据内容，任何类型的数据都可以通过HTTP发送。客户端以及服务器指定使用适合的MIME-type内容类型。

* HTTP是无状态：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。



