### select



调用select函数，select函数会委托内核帮我们做检测，需要给到内核一些数据，这些数据就是select的参数。

参数：

```C
int select(int nfds,        //maxfd+1
           fd_set* readfds, //传入传出参数，读集合
           fd_set*writefds,
           fd_set* exceptfds,
           struct timeval * timeout);

fd_set:是个集合的类型，有1024个标志位；验证方法：sizeof(fd_set)=128，128个字节*8=1024个bit位
NO1参数：遍历的总个数。maxfd是最大编号，从0开始数的话，总个数是maxfd+1;
No2参数：读集合，被动的，并不知道什么时候有数据，委托内核帮助检测；reads是传入传出参数
No3参数：写集合，主动的，想啥时候写就啥时候写
No4参数：异常集合，一般传NULL，特殊需求可以捕捉了看看
No5参数：阻塞时长，设为NULL则为永久阻塞，struct timeval val={1,0};阻塞1s
select(maxfd+1, &read, &write, &expt,timeout);
```

思路：

创建套接字；绑定；监听；

将待检测的数据初始化到fd_set集合中



文件描述符集类型：fd_set rdset;

文件描述符操作函数：

```C
FD_ZERO(fd_set*);//全部清空
FD_SET(fd,fd_set*);//从集合中删除某一项
FD_ISSET(fd,fd_set*);//将某个文件描述符添加到集合，会自动找位置，不需要自己找位置
FD_CLR(fd,fd_set*);//判断某个文件描述符对应的标志位是否为1==判断某个文件描述符是否在集合中
```



循环委托内核区做检测

等待并接受连接请求（此处调用select函数，委托内核帮助检测）

```c
while(1){
    select();
    //判断fd是否是监听的
    //如果不是监听的，那就是已经连接的客户端发送来的数据
    for(){
        fd_isset(i,&reads);//判断i号文件描述符是否在reads里，如果在，说明，这个文件描述符对应的客户端确实给我发数据了；如果不在，就说明没有发数据
    }
}
```



#### 优缺点分析

底层是数组，一旦分配写死了，大小就固定了

优点： 跨平台

缺点：

- 每次调用select，**都需要把fd集合从用户态拷贝到内核态**，这个开销在fd很多时会很大 
- 同时每次调用select，**都需要在内核遍历传递进来的所有fd**，这个开销在fd很多时也很大 
- select**支持的文件描述符数量太小了**，默认是**1024**



#### select多路转接伪代码：

```c
int main(){
    int lfd = socket();
    bind();
    listen();
    
    fd_st reads ,temp;
    fd_zero(&reads);
    fd_set(lfd,&reads);
    
    int maxfd = lfd;
    
    while(1){
        //委托检测
        temp = reads;
        int ret = select(maxfd+1,&reads,NULL,NULL,NULL);//当有内核缓冲区变化，跳出这个语句
        
        //是不是新的连接到达
        if(fd_isset(lfd,&temp)){
            int cfd = accept();//接受新连接
            fd_set(cfd,&reads);//把cfd加入读集合
            maxfd = maxfd>cfd?cfd:maxfd;//更新maxfd
        }
        //客户端发送数据
        for(int i = lfd+1;i<=maxfd;++i){
            if(fd_isset(i,&temp)){
                int len = read();
                if(len==0){//对方断开连接
                    fd_clr(i,&reads);//把断开的客户端从表中删除
                }
                write();
            }
        }      
    }
}
```



#### select多路转接代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>

int main(int argc, const char* argv[])
{
    if(argc < 2)
    {
        printf("eg: ./a.out port\n");
        exit(1);
    }
    struct sockaddr_in serv_addr;
    socklen_t serv_len = sizeof(serv_addr);
    int port = atoi(argv[1]);

    // 创建套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    // 初始化服务器 sockaddr_in 
    memset(&serv_addr, 0, serv_len);
    serv_addr.sin_family = AF_INET;                   // 地址族 
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);    // 监听本机所有的IP
    serv_addr.sin_port = htons(port);            // 设置端口 
    // 绑定IP和端口
    bind(lfd, (struct sockaddr*)&serv_addr, serv_len);

    // 设置同时监听的最大个数
    listen(lfd, 36);
    printf("Start accept ......\n");

    struct sockaddr_in client_addr;
    socklen_t cli_len = sizeof(client_addr);

    // 最大的文件描述符
    int maxfd = lfd;
    // 文件描述符读集合
    fd_set reads, temp;
    // init
    FD_ZERO(&reads);
    FD_SET(lfd, &reads);

    while(1)
    {
        // 委托内核做IO检测
        temp = reads;
        int ret = select(maxfd+1, &temp, NULL, NULL, NULL);
        if(ret == -1)
        {
            perror("select error");
            exit(1);
        }
        // 客户端发起了新的连接
        if(FD_ISSET(lfd, &temp))
        {
            // 接受连接请求 - accept不阻塞
            int cfd = accept(lfd, (struct sockaddr*)&client_addr, &cli_len);
            if(cfd == -1)
            {
                perror("accept error");
                exit(1);
            }
            char ip[64];
            printf("new client IP: %s, Port: %d\n", 
                   inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ip, sizeof(ip)),
                   ntohs(client_addr.sin_port));
            // 将cfd加入到待检测的读集合中 - 下一次就可以检测到了
            FD_SET(cfd, &reads);
            // 更新最大的文件描述符
            maxfd = maxfd < cfd ? cfd : maxfd;
        }
        // 已经连接的客户端有数据到达
        for(int i=lfd+1; i<=maxfd; ++i)
        {
            if(FD_ISSET(i, &temp))
            {
                char buf[1024] = {0};
                int len = recv(i, buf, sizeof(buf), 0);
                if(len == -1)
                {
                    perror("recv error");
                    exit(1);
                }
                else if(len == 0)
                {
                    printf("客户端已经断开了连接\n");
                    close(i);
                    // 从读集合中删除
                    FD_CLR(i, &reads);
                }
                else
                {
                    printf("recv buf: %s\n", buf);
                    send(i, buf, strlen(buf)+1, 0);
                }
            }
        }
    }

    close(lfd);
    return 0;
}
```

