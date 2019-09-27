# Linux的系统编程-进程控制

### 1 进程相关的概念

##### 程序：

编译好的二进制文件；

##### 进程：

- 运行着的程序；
- 程序员的角度：运行一系列指令的过程
- 操作系统的角度：分配系统资源的基本单位

##### 区别：

- 程序占用磁盘，不占用系统资源；内存占用系统资源；
- 一个程序对应多个进程，一个进程对应一个程序；
- 程序没有生命周期，进程有生命周期；



##### 单通道和多通道设计：

时间片轮转，执行速度ns级别，人只能感受到ms级别的变化，宏观上并发，微观上串行



##### CPU和MMU（了解）

存储介质：网络-----硬盘-----内存-----高速缓存cache-----寄存器

CPU：1预取指令-----2译码-----3执行



MMU（位于CPU内）：

1虚拟内存和物理内存的映射

将虚拟内存地址和物理内存地址进行 绑定或叫做映射，比如有个变量i=10存放在物理内存中，先看一下虚拟地址是啥，MMU找到对应的物理地址，取出该数。

2设置修改内存访问级别

0>1>2>3

用户控件映射到物理内存是独立的。

内核区是映射到同一块地址。



##### 进程的状态转换

初始、就绪、运行、挂起、终止

就绪状态----------------------（获得CPU）--------------------运行状态

运行状态-----------------（获得的时间片结束）------------就绪状态

运行状态------（因等待其他资源，主动交出CPU）-----挂起状态

挂起状态-----------（等待的非CPU资源到位了）---------就绪状态

就绪|运行|挂起状态---------------------------------------------终止状态



##### PCB进程控制块

结构体struct task_struct  位于sched.h文件中

- **进程id**，pid_t类型，非负整数
- **进程的状态**，初始、就绪、运行、挂起、停止等状态
- 进程切换时需要保存和恢复的一些CPU寄存器
- 描述虚拟地址空间的信息
- 描述控制终端的信息
- **当前工作目录**
- umask掩码
- 文件描述符表，包含很多指向file结构体的指针
- 和信号相关的信息
- **用户id和组id**
- 会话（Session）和进程组
- 进程可以使用的资源上限  ulimit -a 查看所有资源的上限



##### 环境变量

env查看所有环境变量

`key=val`    =两边都不能有空格

```c++
#include<stdlib.h>
getenv("HOME");//返回一个字符串
setenv//设置环境变量
unsetenv//删除环境变量
```



### 2 fork/getpid/getppid函数的使用



#### fork 创建一个新的进程

```
pid_t fork(void);
```

失败返回-1；

成功两次返回：父进程返回 子进程的id；子进程返回0.



```C++
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
int main(){
    printf("begin...\n");
    pid_t = fork();
    
    //进程创建失败
    if(pid < 0){
        perror("fork err");
        exit(1);
    }
    
    //进程创建成功
    if(pid == 0){
        //子进程
        printf("I am a child，pid=%d，ppid=%d\n",getpid(),getppid());
    }
    else if(pid>0){
        //父进程的逻辑
        printf("childepid=%d,self=%d,ppid=%d\n",pid,getpid(),getppid());
        sleep(1);//让父进程缓1s再死，避免孤儿进程
    }
    printf("End...\n");
    return 0;
}
```

#### 创建n个子进程：

```c++
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

int main(){
    int n=5;
    int i=0;
    pid_t pid = fork();
    for(i=0;i<5;i++){//父进程循环结束
    	pid = fork();
    	if(pid == 0){
    		//son
    		printf("I am son,pid=%d,ppid=%d\n",getpid(),getppid());
    		break;//子进程退出循环的接口
    	}else if(pid>0){
    		//father
    		printf("I am father,pid=%d,ppid=%d\n",getpid(),getppid());
    	}
    }
    
    //尽量精确控制父子进程的退出顺序
    sleep(i);
    if(i<5){
        //son
        printf("I am child,will exit,pid=%d,ppid=%d",getpid(),getppid());
    }else{
        //father
        printf("I am parent,will out pid=%d,ppid=%d\n",getpid(),getppid());
    }
       
    return 0;
}
```



#### 父子进程共享

父子相同点：全局变量、.data、.text、栈、堆、环境变量、用户ID、宿主目录、进程工作目录、信号处理方式

父子不同点：进程ID、fork返回值、父进程ID、进程运行时间、闹钟（定时器）、未决信号集

父子进程之间遵循**读时共享，写时复制**原则。节省内存开销。

父子进程**不共享全局变量**



### 3 ps/kill命令使用

`ps aux` 查看进程信息

`ps ajx`  追溯进程之间的血缘关系   init（进程的祖先）-----sshd-----bash------其他子进程

`kill  -9 2890`： -9是9号信号SIGKILL；2890是进程id

`kill -l`：可以查看所有信号共64个



`ps aux|grep a.out`可以查看当前的进程

`ps aux|grep a.out|grep -v grep |wc -l`返回数字，表示创建了几个子进程

### 4 execl/execlp函数的使用

```
//执行其他程序
int execl(const char*path,const char *arg,...);

//执行程序的时候，使用PATH环境变量，执行的程序可以不用加路径
int execlp(const char*file,const char *arg,...);
```

exec族函数：底层实现：代码段、数据段替换，进程ID啥的都不变，换核不换壳；

```
#include<unistd>
#include<stdio.h>
int main(){

	//execlp("ls","ls","-l","--color=auto",NULL);
	execl("/bin/ls","ls","-l","--color=auto",NULL);

	//不需要判断返回值
	perror("exec err");
	printf("hello\n");
	
	return 0;
}

```

execl实现自定义程序



### 5 什么是孤儿进程 什么是僵尸进程

孤儿进程：父亲死了，子进程被init进程领养。

僵尸进程：子进程死了，父进程没有回收子进程的资源（PCB）。



orphan

```
#include<stdio.h>
#include<unistd.h>
int main(){
	pid_t pid = fork();
	if(pid == 0){
		while(1){
			printf("I am child,pid=%d,ppid=%d\n",getpid(),getppid());
			sleep(1);
		}
	}else if(pid>0){
		printf("I am parent,pid=%d,ppid=%d\n",getpid(),getppid());
		sleep(5);
		printf("I am parent,I will die!\n");
	}
	return 0;
}
```

 此时，ctrl+c已经终止不了程序了，因为父进程已经死了，子进程脱离shell管制；此处应有`kill -9 childpid`



zombie

```c++
#include<stdio.h>
#include<unistd.h>
int main(){
    pid_t pid = fork();
    if(pid==0){
        printf("I am child,pid=%d,ppid=%d\n",getpid(),getppid());
		sleep(5);
		printf("I am child,I will die!\n");
    }else if(pid>0){
        while(1){
            printf("I am parent,pid=%d",getpid());
            sleep(1);
        }
    }
    return 0;
}
```

回收僵尸进程：杀死僵尸进程的父亲，僵尸进程由init领养，负责回收，`kill -9  parentpid`。

僵尸进程是不能用kill直接清除的。



### 6 wait函数的使用

避免产生僵尸进程

```
//回收子进程，知道子进程的死亡原因
pid_t wait(int * status);//status是传出参数；成功返回终止的子进程id，失败返回-1
```

#### wait

作用：

- 阻塞等待 等子进程死亡
- 回收子进程资源
- 查看死亡原因

子进程死亡原因：

- 正常死亡WIFEXITED，为真，使用WEXITSTATUS得到退出状态；
- 非正常死亡WIFSIGNALED，为真，使用WTERMSIG得到信号。

```C++
#include<stdio.h>
#inlcude<unistd.h>
#include<sys/types.h>
#include<sys/wait.h>

int main(){
    pid_t pid = fork();
    if(pid==0){
        printf("I am child,will die!\n");
        sleep(2);
        //while(1){}//通过外部kill杀死，非正常死亡
        //return 101; //正常死亡
        //exit(102); //正常死亡
    }else if(pid>0){
        printf("I am parent,wait for child die!\n");
        
        int status;     
        pid_t wpit = wait(&status);      
        printf("wait ok,wpid=%d,pid=%d\n",wpid,pid);
        
        //正常死亡
        if(WIFEXITED(status)){
            printf("child exit with %d\n",WEXITSTATUS(status));
        }
        //非正常死亡
        if(WIFSIGNALED(status)){
            printf("child killed by %d\n",WTERMSIG(status));
        }
        
        while(1)
        {
            sleep(1);
        }   
    }
    return 0;
}
```

wait回收多个进程：

```
#include<stdio.h>
#inlcude<unistd.h>
#include<sys/types.h>
#include<sys/wait.h>

int main(){
	int n=5;
	int i=0;

    pid_t pid; 
    for(i=0;i<5;i++){
    	pid = fork();
    	if(pid==0){
    		printf("I am child,pid=%d\n",getpid());
    		break;
    	}
    }
    sleep(1);
    if(i==5){
    	for(i=0;i<5;i++){
    		pid_t wpid=wait(NULL);//阻塞，直到回收
    		printf("wpid=%d\n",wpid);//能到这一步说明已经回收了
    	}
    	while(1){
            sleep(1);
        }   
    }
    return 0;
}
```



### 7 waitpid函数的使用

避免产生僵尸进程



#### waitpid

```
//
pid_t waitpid(pid_t pid,int *status,int options);

pid:
<-1 组id
-1 回收任意
0 回收和调度进程组id相同组内的子进程
>0 回收指定的pid

options：
0与wait相同，也会阻塞
WNOHANG如果当前没有子进程退出的，会立刻返回

返回值：
如果设置了WNOHANG，那么如果没有子进程退出，返回0；
如果有子进程退出，返回退出的pid；如果失败返回-1（没有子进程）；
```



waitpid回收进程：

```C++
#include <stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/types.h>
#include<sys/wait.h>

int main(){
	pid_t pid = fork();
	if(pid==0){
		printf("I am child.pid=%d\n",getpid());
		sleep(2);
	}else if(pid>0){
		printf("I am parent,pid=%d\n",getpid());
		int ret;  
		
		while((ret = waitpid(-1,NULL,WNOHANG)) == 0){
            sleep(1);
        }
		printf("ret=%d\n",ret);
        
        ret = waitpid(-1,NULL,WNOHANG);
        if(ret<0){
            perror("wait err");
        }
        
		while(1){
			sleep(1);
		}
	}
	return 0;
}
```



waitpid回收多个进程：

```C++
#include<stdio.h>
#inlcude<unistd.h>
#include<sys/types.h>
#include<sys/wait.h>

int main(){
	int n=5;
	int i=0;

    pid_t pid; 
    for(i=0;i<5;i++){
    	pid = fork();
    	if(pid==0){
    		
    		break;
    	}
    }
    sleep(1);
    if(i==5){
    	//parent 主要用于回收
        printf("I am parent!\n");
        //如何使用waitpid回收？  -1代表子进程都死了，都收了
        while(1){
            pid_t wpid=waitpid(-1,NULL,WNOHANG);//阻塞，直到回收
            if(wpid == -1){
                break;
            }else if(wpid>0){
                printf("wpid=%d\n",wpid);//能到这一步说明已经回收了
            }
        } 
        //保持父进程的活性，看有没有僵尸
        while(1){
            sleep(1);
        }
    	 
    }
    if(i<5){
        printf("I am child,i=%d,pid=%d\n",getpid());
    }
    return 0;
}
```

