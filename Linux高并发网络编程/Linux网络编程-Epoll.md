# epoll

### 函数1

该函数生成一个epoll专用的文件描述符

 size: epoll上能关注的最大描述符数，随便传一个差不过大小的就行；超出会自动扩展，类似vector的自动扩容

```c
 int epoll_create(int size);
```



### 函数2

用于控制某个epoll文件描述符事件，可以注册、修改、删除

epfd: epoll_create生成的epoll专用描述符

op: EPOLL_CTL_ADD -- 注册；  EPOLL_CTL_MOD -- 修改；  EPOLL_CTL_DEL -- 删除 

fd: 关联的文件描述符 

event: 告诉内核要监听什么事件

```C
int epoll_ctl(int epfd, int op, int fd,struct epoll_event *event);
```



fd 的解释：

```C
typedef union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```



event: 告诉内核要监听什么事件 解释：

```C
struct epoll_event { 

	uint32_t events;  

	epoll_data_t data; 

}; 

events：
- EPOLLIN - 读 
- EPOLLOUT - 写 
- EPOLLERR - 异常
```



### 函数3

等待IO事件发生 - 可以设置阻塞的函数

```C
 int epoll_wait(
     int epfd,
     struct epoll_event* events, // 数组
     int maxevents,
     int timeout);
```

epfd: 要检测的句柄 

events：用于回传待处理事件的数组 

maxevents：告诉内核这个events的大小 

timeout：为超时时间 

- -1: 永久阻塞 ，当节点上的 文件描述符对应的 内核缓冲区中 有数据变化的时候，函数返回
- 0: 立即返回 
- 大于0：是多少就是多少毫秒



### 三种工作模式

#### 1 水平触发模式 （LT）

- 只要fd对应的缓冲区有数据，epoll_wait就返回
- 返回的次数与发送数据的次数没有关系
- 是epoll的默认工作模式

#### 2 边沿触发模式 （ET）

- 客户端给server端发数据：发一次数据，server的epoll_wait返回一次
- 不在乎数据是否读完
- 为了提高效率

修改为边沿触发模式其实很简单，将event后面加上一个宏（EPOLLET）就行了

 数据读不完，如何读完？while(recv());数据读完之后recv会阻塞，如何解决阻塞问题

#### 3 边沿非阻塞模式

1. 效率最高
2. 设置非阻塞 

- open()设置flags；必须`O_WDRW|O_NONBLOCK`
- fcntl：`int flag = fcntl(fd,F_GETFL);  flag |= O_NONBLOCK; fcntl(fd,F_SETFL,flag);`
- 将缓冲区的全部数据都读出 `while(recv>0){printf();}` 当缓冲区数据读完之后，返回是否为0？



### 实现伪代码

```C
int main(){

    int lfd = socket();// 创建监听的套接字
    bind();// 绑定
    listen();// 监听

    int epfd = epoll_create(3000);// epoll树根节点
    struct epoll_event all[3000];// 存储发生变化的fd对应信息

    // init
    // 监听的lfd挂到epoll树上
    struct epoll_event ev;
    // 在ev中init lfd信息
    ev.events = EPOLLIN ;
    ev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);

    while(1){
        // 委托内核检测事件
        int ret = epoll_wait(epfd, all, 3000, -1);
        // 根据ret遍历all数组
        for(int i=0; i<ret; ++i)
        {
            int fd = all[i].data.fd;
            // 有新的连接
            if( fd == lfd)
            {
                // 接收连接请求 - accept不阻塞
                int cfd = accept();
                // cfd上树
                ev.events = EPOLLIN;
                ev.data.fd = cfd;
                epoll_ctl(epfd, epoll_ctl_add, cfd, &ev);
            }
            // 已经连接的客户端有数据发送过来
            else
            {
                // 只处理客户端发来的数据
                if(!all[i].events & EPOLLIN)
                {
                    continue;
                }
                // 读数据
                int len = recv();
                if(len == 0)
                {
                    close(fd);
                    // 检测的fd从树上删除
                    epoll_ctl(epfd, epoll_ctl_del, fd, NULL);
                }      
                // 写数据
                send();
            }
        } 
    } 
}
```



#### 完整代码：

```C
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <sys/epoll.h>


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

    // 创建epoll树根节点
    int epfd = epoll_create(2000);
    // 初始化epoll树
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);

    struct epoll_event all[2000];
    while(1)
    {
        // 使用epoll通知内核fd 文件IO检测
        int ret = epoll_wait(epfd, all, sizeof(all)/sizeof(all[0]), -1);

        // 遍历all数组中的前ret个元素
        for(int i=0; i<ret; ++i)
        {
            int fd = all[i].data.fd;
            // 判断是否有新连接
            if(fd == lfd)
            {
                // 接受连接请求
                int cfd = accept(lfd, (struct sockaddr*)&client_addr, &cli_len);
                if(cfd == -1)
                {
                    perror("accept error");
                    exit(1);
                }
                // 将新得到的cfd挂到树上
                struct epoll_event temp;
                temp.events = EPOLLIN;
                temp.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &temp);
                
                // 打印客户端信息
                char ip[64] = {0};
                printf("New Client IP: %s, Port: %d\n",
                    inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ip, sizeof(ip)),
                    ntohs(client_addr.sin_port));
                
            }
            else
            {
                // 处理已经连接的客户端发送过来的数据
                if(!all[i].events & EPOLLIN) 
                {
                    continue;
                }

                // 读数据
                char buf[1024] = {0};
                int len = recv(fd, buf, sizeof(buf), 0);
                if(len == -1)
                {
                    perror("recv error");
                    exit(1);
                }
                else if(len == 0)
                {
                    printf("client disconnected ....\n");
                    close(fd);
                    // fd从epoll树上删除
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                }
                else
                {
                    printf(" recv buf: %s\n", buf);
                    write(fd, buf, len);
                }
            }
        }
    }

    close(lfd);
    return 0;
}
```

#### 完整代码加强版：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <sys/epoll.h>

typedef struct sockinfo
{
    int fd;
    struct sockaddr_in sock;
}SockInfo;

int main(int argc, const char* argv[])
{
    if(argc < 2)
    {
        printf("./a.out port\n");
        exit(1);
    }
    int lfd, cfd;
    struct sockaddr_in serv_addr, clien_addr;
    int serv_len, clien_len;
    int port = atoi(argv[1]);

    // 创建套接字
    lfd = socket(AF_INET, SOCK_STREAM, 0);
    // 初始化服务器 sockaddr_in 
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;                   // 地址族 
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);    // 监听本机所有的IP
    serv_addr.sin_port = htons(port);            // 设置端口 
    serv_len = sizeof(serv_addr);
    // 绑定IP和端口
    bind(lfd, (struct sockaddr*)&serv_addr, serv_len);

    // 设置同时监听的最大个数
    listen(lfd, 36);
    printf("Start accept ......\n");

    // 创建红黑树根节点
    int epfd = epoll_create(2000);
    if(epfd == -1)
    {
        perror("epoll_create error");
        exit(1);
    }
    // lfd添加到监听列表
    SockInfo* sinfo = (SockInfo*)malloc(sizeof(SockInfo));
    sinfo->sock = serv_addr;
    sinfo->fd = lfd;
    struct epoll_event ev;
    ev.data.ptr = sinfo;
    ev.events = EPOLLIN;
    int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl error");
        exit(1);
    }

    struct epoll_event res[2000];
    while(1)
    {            
        // 设置监听
        ret = epoll_wait(epfd, res, sizeof(res)/sizeof(res[0]), -1);
        if(ret == -1)
        {
            perror("epoll_wait error");
            exit(1);
        }

        // 遍历前ret个元素
        for(int i=0; i<ret; ++i)
        {
            int fd = ((SockInfo*)res[i].data.ptr)->fd;
            if(res[i].events != EPOLLIN)
            {
                continue;
            }
            // 判断文件描述符是否为lfd
            if(fd == lfd)
            {
                char ipbuf[64];
                SockInfo *info = (SockInfo*)malloc(sizeof(SockInfo));
                clien_len = sizeof(clien_addr);
                cfd = accept(lfd, (struct sockaddr*)&clien_addr, &clien_len);
                // cfd 添加到监听树
                info->fd = cfd;
                info->sock = clien_addr;
                struct epoll_event ev;
                ev.events = EPOLLIN;
                ev.data.ptr = (void*)info;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                printf("client connected, fd = %d, IP = %s, Port = %d\n", cfd, 
                       inet_ntop(AF_INET, &clien_addr.sin_addr.s_addr, ipbuf, sizeof(ipbuf)), 
                       ntohs(clien_addr.sin_port));
            }
            else
            {
                // 通信
                char ipbuf[64];
                char buf[1024] = {0};
                int len = recv(fd, buf, sizeof(buf), 0);
                SockInfo* p = (SockInfo*)res[i].data.ptr;
                if(len == -1)
                {
                    perror("recv error");
                    exit(1);
                }
                else if(len == 0)
                {
                    // ip
                    inet_ntop(AF_INET, &p->sock.sin_addr.s_addr, ipbuf, sizeof(ipbuf));
                    printf("client %d 已经断开连接, Ip = %s, Port = %d\n", 
                           fd, ipbuf, ntohs(p->sock.sin_port));
                    // 节点从树上删除
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                    free(p);
                }
                else
                {
                    printf("Recv data from client %d, Ip = %s, Port = %d\n", 
                           fd, ipbuf, ntohs(p->sock.sin_port));
                    printf("    === buf = %s\n", buf);
                    send(fd, buf, strlen(buf)+1, 0);
                }
            }
        }
    }

    close(lfd);
    free(sinfo);
    return 0;
}
```

