Linux网络编程-本地套接字



## 1 文件格式

管道：p

套接字：s

伪文件，不管写什么，大小都是0，其实是保存在内核缓冲区   



## 2 服务器端

创建套接字：socket（AF_LOCKAL，sock_stream,0);

绑定：

```C
struct sockaddr_un serv;
serv.sun_family = af_local;
strcpy(serv.sun_path,"server.socket");//现在还不存在
bind(lfd,(struct sockaddr8)&serv,len);//绑定成功套接字文件被创建
```

设置监听：

```c
listen();
```

等待接收连接请求：

```c
struct sockaddr_un client;
int len = sizeof(client);
int cfd = accept(lfd,&client,&len);
```

通信：

```c
send
recv
```

断开连接：

```c
close(cfd);
close(lfd);
```



代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/un.h>

int main(int argc, const char* argv[])
{
    int lfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 如果套接字文件存在, 删除套接字文件
    unlink("server.sock");

    // 绑定
    struct sockaddr_un serv;
    serv.sun_family = AF_LOCAL;
    strcpy(serv.sun_path, "server.sock");
    int ret = bind(lfd, (struct sockaddr*)&serv, sizeof(serv));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }
     
    // 监听
    ret = listen(lfd, 36);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 等待接收连接请求
    struct sockaddr_un client;
    socklen_t len = sizeof(client);
    int cfd = accept(lfd, (struct sockaddr*)&client, &len);
    if(cfd == -1)
    {
        perror("accept error");
        exit(1);
    }
    printf("======client bind file: %s\n", client.sun_path);
     
    // 通信
    while(1)
    {
        char buf[1024] = {0};
        int recvlen = recv(cfd, buf, sizeof(buf), 0);
        if(recvlen == -1)
        {
            perror("recv error");
            exit(1);
        }
        else if(recvlen == 0)
        {
            printf("clietn disconnect ....\n");
            close(cfd);
            break;
        }
        else
        {
            printf("recv buf: %s\n", buf);
            send(cfd, buf, recvlen, 0);
        }
    }
    close(cfd);
    close(lfd);
    
    return 0;
}
```



### 3 客户端

创建套接字：

```c
int fd = socket(af_local,sock_stream,0);
```

绑定一个套接字文件：

```c
struct sockaddr_un client;
client.sun_family = af_local;
strcpy(client.sun_path,"client.socket");//现在还不存在
bind(fd,(struct sockaddr8)&serv,len);//绑定成功套接
```

连接服务器

```c
struct sockaddr_un serv;
serv.sun_family = af_local;
strcpy(serv.sun_path,"server.socket");//现在还不存在
connect(fd,&serv,sizeof(server));
```

 通信：

```c
send
recv
```

关闭：

```c
close
```



代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/un.h>

int main(int argc, const char* argv[])
{
    int fd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket error");
        exit(1);
    }

    unlink("client.sock");

    // ================================
    // 给客户端绑定一个套接字文件
    struct sockaddr_un client;
    client.sun_family = AF_LOCAL;
    strcpy(client.sun_path, "client.sock");
    int ret = bind(fd, (struct sockaddr*)&client, sizeof(client));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 初始化server信息
    struct sockaddr_un serv;
    serv.sun_family = AF_LOCAL;
    strcpy(serv.sun_path, "server.sock");

    // 连接服务器
    connect(fd, (struct sockaddr*)&serv, sizeof(serv));

    // 通信
    while(1)
    {
        char buf[1024] = {0};
        fgets(buf, sizeof(buf), stdin);
        send(fd, buf, strlen(buf)+1, 0);

        // 接收数据
        recv(fd, buf, sizeof(buf), 0);
        printf("recv buf: %s\n", buf);
    }

    close(fd);

    return 0;
}
```

