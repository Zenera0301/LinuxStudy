---
title: Linux的系统编程-进程通信
---

IPC：进程间通信，通过内核提供的缓冲区进行数据交换的机制。

方式：

1. **pipe 管道---最简单**
2. **fifo 有名管道**
3. **mmap 文件映射共享IO---速度最快**
4. 本地socket 最稳定
5. **信号**---携带信息量最小
6. 共享内存---用特定API申请的一块内存，进程退出，其他进程仍可访问这块内存
7. 消息队列



### 1 使用pipe进行父子进程间通信

```c++
#include <unistd.h>
int pipe(int pipefd[2]);//pipefd文件读写描述符，0---读；1---写；返回值：成功返回0，失败返回-1
```

常见通信方式：单工（广播）；半双工（同一个时刻，数据只能往一个方向发，对讲机）；全双工（打电话）。

管道：半双工

#### 父子进程通信：

```C++
#include <stdio.h>
#include <unistd.h>
int main(){
    int fd[2];
    pipe(fd);
    pid_t pid=fork();
    if(pid==0){
        //son
        write(fd[1],"hello",5);
    }else if(pid>0){
        //parent
        char buf[12]={0};
        int ret = read(fd[0],buf,sizeof(buf));
        if (ret>0){
            write(STDOUT_FILENO,buf,ret);
        }
    }
    return 0;
}
```



#### 管道实例：`ps aux|grep bash`

```C
#include <stdio.h>
#include <unistd.h>

int main()
{
    int fd[2];
    pipe(fd);

    pid_t pid = fork();
    if(pid == 0){
        //son 
        //son -- > ps 
        //关闭 读端
        close(fd[0]);
        //1. 先重定向
        dup2(fd[1],STDOUT_FILENO);//标准输出重定向到管道写端（获取输出的内容，写入管道）
        //2. execlp 
        execlp("ps","ps","aux",NULL);
    }else if(pid > 0){
        //parent
        //关闭写端
        close(fd[1]);
        //1. 先重定向，标准输入重定向到管道读端(读取输入的内容)
        dup2(fd[0],STDIN_FILENO);
        //2. execlp 
        execlp("grep","grep","bash",NULL);
    }
    return 0;
}
```

#### 读管道：

1. 若写端全部关闭---read读到0，相当于读到文件末尾

2. 若写端没有全部关闭

   有数据---read读到数据

   没有数据--read阻塞，fcntl函数可以更改非阻塞

#### 写管道：

1. 若读端全部关闭---会产生一个13号信号SIGPIPE，程序异常终止；

2. 若读端未全部关闭

   管道已满---write阻塞

   管道未满---write正常写入

#### 计算管道大小：

​	`ulimit -a` 查看上限512*8

​	`long fpathconf(int fd,int name);`

#### 管道优劣：

- 优点：简单，相比信号，套接字实现进程间通信，简单很多。
- 缺点：1. 只能单向通信，双向通信需建立两个管道。

​	     	      2. 只能用于**父子、兄弟**进程(有共同祖先)间通信。该问题后来使用fifo有名管道解决。





### 2 使用pipe进行兄弟进程间通信

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main()
{
    int fd[2];
    pipe(fd);
    pid_t pid;
    int n =2,i = 0;
    //创建两个子进程
    for(i = 0; i < n ; i ++){
        pid = fork();
        if(pid == 0){
            break;//不让子进程创建子进程
        }
    }

    //i = 0 ,代表兄长，1 - 代表弟弟，2- 父亲
    if(i == 0){
        //兄长进程
        //1. 关闭读端
        close(fd[0]);
        //2. 重定向
        dup2(fd[1],STDOUT_FILENO);
        //3. 执行 execlp
        execlp("ps","ps","aux",NULL);
            
    }else if(i == 1){
        //弟弟 
        //1. 关闭写端
        close(fd[1]);
        //2. 重定向
        dup2(fd[0],STDIN_FILENO);
        //3. 执行ececlp
        execlp("grep","grep","bash",NULL);

    }else if(i == 2){
        //parent 
        //父亲需要关闭读写两端
        close(fd[0]);
        close(fd[1]);
        //回收子进程
        wait(NULL);
        wait(NULL);
    }
    return 0;
}
```





### 3 使用fifo进行无血缘关系的进程间通信

FIFO 有名管道，实现无血缘关系间的管道通信

通过队列实现，**内核**会针对fifo文件开辟一个**缓冲区**，**操作fifo文件，可以操作缓冲区**，实现进程间通信--实际上就是**文件读写**。

```
$ mkfifo myfifo 命令创建
或
int mkfifo(const char *pathname, mode_t mode);
```

fifo读

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>

int main(int argc,char *argv[])
{
    if(argc != 2){
        printf("./a.out fifoname\n");
        return -1;
    }
    printf("begin oepn read...\n");
    int fd = open(argv[1],O_RDONLY);
    printf("end oepn read...\n");
    
    char buf[256];
    int ret;
    while(1){
        //循环读
        memset(buf,0x00,sizeof(buf));
        ret = read(fd,buf,sizeof(buf));
        if(ret > 0){
            printf("read:%s\n",buf);
        }
    }

    close(fd);
    return 0;
}
```

fifo写

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>

int main(int argc,char * argv[])
{
    if(argc != 2){
        printf("./a.out fifoname\n");
        return -1;
    }
    
    // 当前目录有一个 myfifo 文件
    //打开fifo文件
    printf("begin open ....\n");
    int fd = open(argv[1],O_WRONLY);
    printf("end open ....\n");
    //写
    char buf[256];
    int num = 1;
    while(1){
        memset(buf,0x00,sizeof(buf));
        sprintf(buf,"xiaoming%04d",num++);
        write(fd,buf,strlen(buf));
        sleep(1);
        //循环写
    }
    //关闭描述符
    close(fd);
    return 0;
}
```

> open注意：打开fifo文件的时候，read端会阻塞等待write端open，同理，write端也会阻塞等待read端open。





### 4 mmap函数的使用

把文件中的一段内容映射到内存中的映射区，操作内存

```C
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,int fd, off_t offset);

int munmap(void *addr, size_t length);
```

**创建映射区**

**void *mmap(void *addr, size_t length, int prot, int flags,int fd, off_t offset);**

○ addr 传NULL 

○ length 映射区的长度 

○ prot 

- PROT_READ 可读 
- PROT_WRITE 可写 

○ flags 

- MAP_SHARED **共享的，对内存的修改会影响到源文件** 
- MAP_PRIVATE 私有的 

○ fd 文件描述符，open打开一个文件 

○ offset 偏移量 

- 成功 返回 可用的内存首地址 
- 失败 返回 MAP_FAILED



**释放映射区** 

**int munmap(void *addr, size_t length);**

○ addr 传mmap的返回值 

○ length mmap创建的长度 

- int munmap(void *addr, size_t length); 
- mmap共享映射区 

○ 返回值

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

int main()
{
    int fd = open("mem.txt",O_RDWR);//创建并且截断文件
    //int fd = open("mem.txt",O_RDWR|O_CREAT|O_TRUNC,0664);//创建并且截断文件

    ftruncate(fd,8);   
    //创建映射区
   char *mem = mmap(NULL,20,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    //char *mem = mmap(NULL,8,PROT_READ|PROT_WRITE,MAP_PRIVATE,fd,0);

    if(mem == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
    close(fd);
    //拷贝数据
    strcpy(mem,"helloworld");
//    mem++;
    //释放mmap
    if(munmap(mem,20) < 0){
        perror("munmap err");
    }
    return 0;
}
```



#### mmap九问： 

1. 如果更改mem变量的地址，释放的时候munmap，传入mem还能成功吗？不能！！ 

2. 如果对mem越界操作会怎么样？**文件的大小对映射区操作有影响，尽量避免。** 

3. 如果文件偏移量随便填个数会怎么样？**offset必须是 4k的整数倍** 

4. 如果文件描述符先关闭，对mmap映射有没有影响？没有影响 

5. open的时候，可以新创建一个文件来创建映射区吗？不可以用大小为0的文件 

6. open文件选择O_WRONLY，可以吗？ 不可以： Permission denied 

7. 当选择MAP_SHARED的时候，open文件选择O_RDONLY，prot可以选择 PROT_READ|PROT_WRITE吗？Permission denied ，SHARED的时候，映射区的权限<=  open文件的权限 

8. mmap什么情况下会报错？很多情况 

9. 如果不判断返回值会怎么样？ 会死的很难堪！！

   

### 5 mmap函数创建匿名映射区的方法

#### 父子进程通信(利用匿名映射区)：

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/wait.h>

int main()
{
    //创建内存映射区
    int *mem = mmap(NULL,4,PROT_READ|PROT_WRITE,MAP_SHARED|MAP_ANON,-1,0);
	
    //是否创建成功
    if(mem == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
	
    //创建子进程
    pid_t pid = fork();

    //针对父子进程处理
    if(pid == 0 ){
        //son 
        *mem = 101;
        printf("child,*mem=%d\n",*mem);
        sleep(3);
        printf("child,*mem=%d\n",*mem);
    }else if(pid > 0){
        //parent 
        sleep(1);
        printf("parent,*mem=%d\n",*mem);
        *mem = 10001;
        printf("parent,*mem=%d\n",*mem);
        wait(NULL); //回收子进程
    }

    munmap(mem,4);
    return 0;
}
```



### 6 mmap函数进行有血缘关系的进程间通信

#### 父子进程通信（利用文件）：缺点：用了一个文件，并没用到文件内容

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/wait.h>

int main()
{
    // 先创建映射区
    int fd = open("mem.txt",O_RDWR);
    //int *mem = mmap(NULL,4,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    int *mem = mmap(NULL,4,PROT_READ|PROT_WRITE,MAP_PRIVATE,fd,0);
    
    if(mem == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
    
    // fork子进程
    pid_t pid = fork();

    // 父进程和子进程交替修改数据
    if(pid == 0 ){
        //son 
        *mem = 100;
        printf("child,*mem = %d\n",*mem);
        sleep(3);
        printf("child,*mem = %d\n",*mem);
    }
    else if(pid > 0){
        //parent
        sleep(1);
        printf("parent,*mem=%d\n",*mem);
        *mem = 1001;
        printf("parent,*mem=%d\n",*mem);
        wait(NULL);//回收子进程
    }

    munmap(mem,4);
    close(fd);
    return 0;
}
```



### 7 mmap函数进行无血缘关系的进程间通信

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/wait.h>

typedef struct  _Student{
    int sid;
    char sname[20];
}Student;

int main(int argc,char *argv[])
{
    if(argc != 2){
        printf("./a.out filename\n");
        return -1;
    }
    
    // 1. open file 
    int fd = open(argv[1],O_RDWR|O_CREAT|O_TRUNC,0666);
    int length = sizeof(Student);

    ftruncate(fd,length);

    // 2. mmap
    Student * stu = mmap(NULL,length,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    
	//判断返回值
    if(stu == MAP_FAILED){
        perror("mmap err");
        return -1;
    }
    int num = 1;
    // 3. 修改内存数据
    while(1){
        stu->sid = num;
        sprintf(stu->sname,"xiaoming-%03d",num++);
        sleep(1);//相当于没隔1s修改一次映射区的内容
    }
    // 4. 释放映射区和关闭文件描述符
    munmap(stu,length);
    close(fd);

    return 0;
}
```



