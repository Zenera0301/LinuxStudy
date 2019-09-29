# Socket

程序员要做的事：操作文件（内核缓冲区）



工作模式：

- 创建一个文件描述符fd
- 这个fd对应一块内核缓冲区
- 这个内核缓冲区是分为两部分的：一部分专门用于读；一部分专门用于写
- read的时候是读内核缓冲区中的数据；write的时候是往这块内核缓冲区写数据
- 内核缓冲区负责写的部分若有数据，数据就直接发送出去了



### 服务器流程：

1. 创建套接字 

   ```C
   int lfd = socket;
   ```

2. 绑定本地IP和端口

   ```C
   struct socketaddr_in serv;
   serv.port = htons(port); // 小端转大端，主机字节序转网络字节序
   serv.IP = htons(INADDR_ANY);// 小端转大端，主机字节序转网络字节序
   bind(lfd,&serv,sizeof(serv));
   ```

3. 监听

   ```C
   listen(lfd,128);//128是同时连接到服务器上的个数
   ```

4. 等待并接收连接请求

   ```C
   //创建一个结构体，用于保存：连接到服务器上的客户端的IP和端口
   struct sockaddr_in client;
   int len = sizeof(client);
   int len = sizeof(client);
   int cfd = accept(lfd,&client,&len); //cfd 是用于通信的
   ```

5. 通信

   ```C
   read/recv  //接收数据
   write/send //发送数据
   ```

6. 关闭

   ```C
   close(lfd);//用于监听的文件描述符
   close(cfd);//用于通信的文件描述符
   ```

   linux 和windows的区别，Windows 中用recv/send，需要加载套接字库才能用，结束后，把套接字库关闭



### 客户端流程

1. 创建套接字

   ```C
   int fd = socket;
   ```

2. 连接服务器

   ```C
   struct sockaddr_in server;
   server.port
   server.ip = (int)...
   server.family
   connect(fd,&server,sizeof(server));
   ```

3. 通信

   ```C
   read/recv  //接收数据
   write/send //发送数据
   ```

4. 断开连接

   ```C
   close(fd);
   ```

   

