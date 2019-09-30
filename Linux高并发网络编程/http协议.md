# http

应用层协议

### 1 请求消息（Request），浏览器给服务器发送

http协议包括四部分：

1. **请求行**：请求类型，要访问的资源，使用的http版本，例如：  

   `GET /1.txt  HTTP/1.1`  和`POST HTTP/1.1`

2. **请求头**：服务器要使用的附加信息（由键值对组成）

   ```
   Host: localhost:2222
   User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:24.0) Gecko/201001 01 
   Firefox/24.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
   Accept-Language: zh-cn,zh;q=0.8,en-us;q=0.5,en;q=0.3
   Accept-Encoding: gzip, deflate
   Connection: keep-alive
   If-Modified-Since: Fri, 18 Jul 2014 08:36:36 GMT
   ```

3. **空行**：空行是必须要有的，即使没有请求数据，就是换行：\r\n

4. **请求数据**：主体，可以添加任意的其他数据（可以有，可以没有）





### 2 响应消息（Response），服务器收到请求后给浏览器发

响应消息包括四部分：

1. **状态行**：包括http协议版本号，状态码，状态信息
2. **消息报头**：说明客户端要使用的一些附加信息
3. **空行**：空行是必须要有的
4. **响应正文**：服务器返回给客户端的文本信息

```
HTTP/1.1 200 Ok
Server: micro_httpd
Date: Fri, 18 Jul 2014 14:34:26 GMT
Content-Type: text/plain; charset=iso-8859-1 (必选项,告诉浏览器发送的数据是什么类型)
Content-Length: 32(发送的数据的长度，或者写-1，或者不写)
Content-Language: zh-CN
Last-Modified: Fri, 18 Jul 2014 08:36:36 GMT
Connection: close

#include <stdio.h>
int main(void){
    printf("hello world!\n");
    return 0;
}
```



### 3 Get和Post的区别

#### Get：

- 请求数据的时候，用户名和密码等敏感信息会显示在地址栏中，不够安全，不适用敏感信息
- url的长度有限，不同浏览器限制不同，能显示的消息有限，适用于小数据量
- get产生一个包，header和request body放在一起发送出去，服务器收到后返回200 ok

#### Post：

- 提交数据的时候，数据不显示在地址栏里，隐藏起来的，相对更安全，提交敏感信息用post
- post对于文件大小没有限定，适合大数据量
- post产生两个包：一个header，一个request body；发送header后，服务器返回100 continue；发送body后，服务器返回200 ok





### 4 浏览器使用get方法向服务器请求数据：

浏览器地址栏：192.168.1.115/hello.c

浏览器封装一个http请求协议：

```
get /hello.c http/1.1
key:value
key:value
key:value
key:value
\r\n
```

服务器解析这个请求：

分析请求行，根据资源目录，找到所需资源hello.c，用open打开文件，read读取文件内容，发送给浏览器





### 5  HTTP常用状态码 

状态代码由3位数字组成，第一个数字定义了响应的类别，共分5种类别: 

○ 1xx：**指示信息**--表示请求已接收，继续处理 

○ 2xx：**成功**--表示请求已被成功接收、理解、接受 

○ 3xx：**重定向**--要完成请求必须进行更进一步的操作 

○ 4xx：**客户端错误**--请求有语法错误或请求无法实现 

○ 5xx：**服务器端错误**--服务器未能实现合法的请求 

```
常见状态码：
 200 OK 客户端请求成功
 400 Bad Request 客户端请求有语法错误，不能被服务器所理解
 401 Unauthorized 请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
 403 Forbidden 服务器收到请求，但是拒绝提供服务
 404 Not Found 请求资源不存在，eg：输入了错误的URL
 500 Internal Server Error 服务器发生不可预期的错误
 503 Server Unavailable 服务器当前不能处理客户端的请求，一段时间后可能恢复正常
```

