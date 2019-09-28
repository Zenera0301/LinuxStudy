---
title: Socket通信
---

程序员要做的事：文件操作（内核缓冲区）

### 1 socket tcp server服务器端的流程

- 创建套接字**socket()**

- 绑定本地IP和端口**bind()**

- 监听**listen()**

- 等待并接收连接请求**accept()**


- 通信

  接收数据**read / recv**

  发送数据**write / send**

  linux 和windows的区别，Windows 中用recv/send，需要加载套接字库才能用，结束后，把套接字库关闭

- 关闭**close()**






### 2 客户端

创建套接字 

int fd = socket



连接服务器

struct sockaddr_in server;

server.port

server.ip = (int)...

server.family

connect (fd,&server,sizeof(server));



通信

接收数据：read/recv

发送数据：write/send



断开连接

close(fd);

操作IP和Port的时候需要将 主机字节序转网络字节序（小端转大端）；接收和发送数据的时候不需要转换。



### 3 tcp多进程并发服务器

 只能处理单链接

- 创建套接字--监听
- 绑定--bind
- 监听--listen(lfd,128)
- 接受连接请求---------------------------------父进程sever（默认只有一个进程，认为是父进程）
- 通信



#### 多进程版的并发服务器：

1. 父进程只负责等待并且接受连接请求，写到while循环中；
2. 一旦有连接请求，父进程就fork()出来一个子进程，得到文件描述符，让子进程拿到这个文件描述符去和文件描述符对应的客户端去通信。

#### 使用多进程的方式, 解决服务器处理多连接的问题:

**1.共享**

**○ 读时共享, 写时复制** （父进程和子进程都去同一个物理内存中，读取同一个东西，写的时候各写各的）

读时共享：在父进程里定义了一个变量，子进程里照样有这个变量；

 写时复制：修改父进程，对子进程没有影响；修改子进程1，对子进程2也没有影响。

**○ 文件描述符** 

**○ 内存映射区** -- 通过mmap创建出来的 

mmap能够创建内存映射区，能够实现进程间通信，文件操作，文件复制等



**2. 父进程 的角色是什么?** 

1. 用accept()等待接受客户端连接，有连接的时候accept就不堵塞了，这时用fork()创建一个子进程;
2. 将用于通信的文件描述符关掉；（避免资源浪费）

**3. 子进程的角色是什么?** 

1. 通信，使用accept返回值fd，拿到文件描述符（由于读时共享，这个文件描述符与父进程中得到的文件描述符相同），就可以对应的客户端做通信了；
2. 关掉用于监听的文件描述符；（避免资源浪费）

**4. 创建的进程的个数有限制吗?** 

1. 受硬件控制
2. 文件描述符默认也是有上限的1024 



**5. 子进程资源回收** 

1. 避免孤儿进程和僵尸进程的出现
2. 调用函数回收：wait/waitpid
3. 用信号回收：signal,sigaction-推荐



#### 多进程的伪代码

```
//头文件


//回调函数，用于进程回收
void recyle(int num){
    while(waitpid(-1,NULL,wnohang)>0);
}

//主函数
int main(){

//创建套接字
int lfd=sock();

//绑定
bind();

//设置监听
listen();

//信号回收子进程
struct sigaction act;
act.sa_handler = recyle;//回调函数
act.sa_falgs=0;
sigemptyset(&act.sa_mask);
sigaction(SIGCHID,&act,NULL);

//父进程
while(1){
//判断有没有客户端连接
int cfd = accept();//如果有连接，accept就有返回了，否则阻塞在这，停住
//说明已经建立新的连接
//创建子进程
pid_t pid=fork();
//子进程
if(pid==0){
	close(lfd);
	//通信
	while(1){
		int len = read();
		if(len==-1){
			exit(1);//退出
		}else if(len==0){//说明对方关闭了连接了
			close(cfd);
			break;
		}else{
			write(); 
		}
	}
	//退出子进程
	return 0；//或者exit(1);
}else{
	//父进程
	close(cfd);
	while(waitpid(-1,NULL,wnohang)!=-1);
}
}
}
```

完整的多进程实现：

```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <signal.h>
#include <sys/wait.h>
#include <errno.h>

// 进程回收函数
void recyle(int num)
{
    pid_t pid;
    while( (pid = waitpid(-1, NULL, WNOHANG)) > 0 )
    {
        printf("child died , pid = %d\n", pid);
    }
}

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

    // 使用信号回收子进程pcb
    struct sigaction act;
    act.sa_handler = recyle;
    act.sa_flags = 0;
    sigemptyset(&act.sa_mask);
    sigaction(SIGCHLD, &act, NULL);

    struct sockaddr_in client_addr;
    socklen_t cli_len = sizeof(client_addr);
    while(1)
    {
        // 父进程接收连接请求
        // accept阻塞的时候被信号中断, 处理信号对应的操作之后
        // 回来之后不阻塞了, 直接返回-1, 这时候 errno==EINTR
        int cfd = accept(lfd, (struct sockaddr*)&client_addr, &cli_len);
        while(cfd == -1 && errno == EINTR)
        {
            cfd = accept(lfd, (struct sockaddr*)&client_addr, &cli_len);
        }
        printf("connect sucessful\n");
        // 创建子进程
        pid_t pid = fork();
        if(pid == 0)
        {
            close(lfd);
            // child process
            // 通信
            char ip[64];
            while(1)
            {
                // client ip port
                printf("client IP: %s, port: %d\n", 
                       inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ip, sizeof(ip)),
                       ntohs(client_addr.sin_port));
                char buf[1024];
                int len = read(cfd, buf, sizeof(buf));
                if(len == -1)
                {
                    perror("read error");
                    exit(1);
                }
                else if(len == 0)
                {
                    printf("客户端断开了连接\n");
                    close(cfd);
                    break;
                }
                else
                {
                    printf("recv buf: %s\n", buf);
                    write(cfd, buf, len);
                }
            }
            // 干掉子进程
            return 0;

        }
        else if(pid > 0)
        {
            // parent process
            close(cfd);
        }
    }

    close(lfd);
    return 0;
}
```

启用server和client后，客户端给服务器端发送heloo：

server：

```
dj@dj:~/dingjing$ gcc process_server.c -o server
dj@dj:~/dingjing$ ./server 9876
Start accept ......
connect sucessful
client IP: 127.0.0.1, port: 35394
recv buf: heloo

client IP: 127.0.0.1, port: 35394

```

client：

```
dj@dj:~$ nc 127.1 9876
heloo
heloo
```



### 4.tcp多线程并发服务器

线程共享：全局数据区的数据，堆区数据，一块有效内存的地址

主线程做的事情和主进程类似

伪代码：

```
//头文件


//回调函数，用于进程回收
void* worker(void* arg){
    while(1){
        //打印客户端ip和port
    }
}
//定义结构体
typedef struct sockInfo{
	pthread_t id;//定义线程号
    int fd;//文件描述符
    struct sockaddr_in addr;//定义一个结构体变量
}SockInfo;

//主函数
int main(){

    //创建套接字
    int lfd=sock();

    //绑定
    bind();

    //设置监听
    listen();

    sockInfo sock[256];
    //父线程
    while(1){

        //判断有没有客户端连接
        sock[i].fd = accept(lfd,&client,&len);//如果有连接，accept就有返回了，否则阻塞在这，停住

        //说明已经建立新的连接
        //创建子线程
        pthread_creat(&sock[i].id,NULL,worker,&sock[i]);//第一个连接
        
        //回收子线程
        pthread_deatch(sock[i].id);
	}
}
```





线程需要去对应一些独立的数据；存储到一块独立的内存里，就独立了；定义一个全局数组。

进程就不需要，因为，进程创建完就是各自独立的。



完整的多线程实现：

```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <pthread.h>

// 自定义数据结构
typedef struct SockInfo
{
    int fd; // 
    struct sockaddr_in addr;
    pthread_t id;
}SockInfo;

// 子线程处理函数
void* worker(void* arg)
{
    char ip[64];
    char buf[1024];
    SockInfo* info = (SockInfo*)arg;
    // 通信
    while(1)
    {
        printf("Client IP: %s, port: %d\n",
               inet_ntop(AF_INET, &info->addr.sin_addr.s_addr, ip, sizeof(ip)),
               ntohs(info->addr.sin_port));
        int len = read(info->fd, buf, sizeof(buf));
        if(len == -1)
        {
            perror("read error");
            pthread_exit(NULL);//只退出单个子进程，不影响客户端连接
        }
        else if(len == 0)
        {
            printf("客户端已经断开了连接\n");
            close(info->fd);
            break;
        }
        else
        {
            printf("recv buf: %s\n", buf);
            write(info->fd, buf, len);
        }
    }
    return NULL;
}

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

    int i = 0;
    SockInfo info[256];
    // 规定 fd == -1  
    for(i=0; i<sizeof(info)/sizeof(info[0]); ++i)
    {
        info[i].fd = -1;
    }

    socklen_t cli_len = sizeof(struct sockaddr_in);
    while(1)
    {
        // 选一个没有被使用的, 最小的数组元素
        for(i=0; i<256; ++i)
        {
            if(info[i].fd == -1)
            {
                break;
            }
        }
        if(i == 256)
        {
            break;
        }
        // 主线程 - 等待接受连接请求
        info[i].fd = accept(lfd, (struct sockaddr*)&info[i].addr, &cli_len);

        // 创建子线程 - 通信
        pthread_create(&info[i].id, NULL, worker, &info[i]);
        // 设置线程分离
        pthread_detach(info[i].id);

    }

    close(lfd);

    // 只退出主线程
    pthread_exit(NULL);
    return 0;
}
```

