### UDP通信流程

#### 服务器端

1. 创建套接字： `int fd = socket(af_inet,sock_dgram,0);`

2. 绑定本地IP和端口：`bind();`

3. 通信：

   接收数据`recvfrom`；会保存client的IP和端口

   发送数据`sendto`

4. 关闭套接字：`close(fd);`

```C
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>

int main(int argc, const char* argv[])
{
    // 创建套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket error");
        exit(1);
    }
    
    // fd绑定本地的IP和端口
    struct sockaddr_in serv;
    memset(&serv, 0, sizeof(serv));
    serv.sin_family = AF_INET;
    serv.sin_port = htons(8765);
    serv.sin_addr.s_addr = htonl(INADDR_ANY);
    int ret = bind(fd, (struct sockaddr*)&serv, sizeof(serv));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    struct sockaddr_in client;
    socklen_t cli_len = sizeof(client);
    // 通信
    char buf[1024] = {0};
    while(1)
    {
        int recvlen = recvfrom(fd, buf, sizeof(buf), 0,                         									(struct sockaddr*)&client, &cli_len);
        if(recvlen == -1)
        {
            perror("recvform error");
            exit(1);
        }
        
        printf("recv buf: %s\n", buf);
        char ip[64] = {0};
        printf("New Client IP: %s, Port: %d\n",
            inet_ntop(AF_INET, &client.sin_addr.s_addr, ip, sizeof(ip)),
            ntohs(client.sin_port));

        // 给客户端发送数据
        sendto(fd, buf, strlen(buf)+1, 0, (struct sockaddr*)&client, sizeof(client));
    }
    
    close(fd);

    return 0;
}
```



#### 客户端

1. 创建套接字： `int fd = socket(af_inet,sock_dgram,0);`
2. 通信：接收数据`recvfrom`；发送数据`sendto`
3. 关闭套接字：`close(fd);`

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>

int main(int argc, const char* argv[])
{
    // create socket
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 初始化服务器的IP和端口
    struct sockaddr_in serv;
    memset(&serv, 0, sizeof(serv));
    serv.sin_family = AF_INET;
    serv.sin_port = htons(8765);
    inet_pton(AF_INET, "127.0.0.1", &serv.sin_addr.s_addr);

    // 通信
    while(1)
    {
        char buf[1024] = {0};
        fgets(buf, sizeof(buf), stdin);
        // 数据的发送 - server - IP port
        sendto(fd, buf, strlen(buf)+1, 0, (struct sockaddr*)&serv, sizeof(serv));

        // 等待服务器发送数据过来
        recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("recv buf: %s\n", buf);
    }
    
    close(fd);

    return 0;
}
```

