# 进程学习

#### 进程和线程

进程：车间

线程：车间里的工人

 

线程概念：轻量级的进程，一个进程内部可以有多个线程，默认情况下的一个进程只有一个线程；

在Linux下，线程是最小的执行单位，进程是最小的系统资源分配单位；

从内核看，线程和进程是一样的，都是通过clone函数实现的，控件共享--线程，空间不共享--进程；

线程有自己的PCB表；



#### 线程的非共享资源

1. 线程id

2. 处理器现场和栈指针（**内核栈**）

3. 独立的栈空间（**用户空间栈**）

4. errno变量

5. **信号屏蔽字**

6. 调度优先级

   

#### 线程的共享资源

1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户ID和组ID
5. 内存地址空间（除了栈之外的：text/data/bss/heap/共享库）



#### 线程的优点

- 提高并发性
- 占用资源小
- 数据通信、共享数据方便



#### 线程的缺点

- 调试困难
- 库函数，不稳定
- 对信号支持不好



#### 线程创建 create

```C
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,                          void *(*start_routine) (void *), void *arg);
//Compile and link with -pthread.
```

1. thread :线程的id，传出参数
2. attr :线程的属性
3. 第三个参数，函数指针，void* func(void*)
4. arg :线程执行函数的参数

返回值：成功返回0，失败返回errno

编译的时候需要加pthread库

```C
#include <stdio.h>
#include<unisted.h>
#include<pthread.h>
void * thr(void *arg){
    printf("I am a thread! pid=%d,tid=%lu\n",getpid(),pthread_self());
    return NULL;
}

int main(){
    pthread_t tid;
    pthread_create(&tid,NULL,thr,NULL);
    printf("I am a main thread,pid=%d,tid=%d\n",getpid(),pthread_self());
    pthread_exit(NULL);
    return 0;
}

//运行命令：
//gcc test.c -lpthread
```





#### 线程退出 exit

```
void pthread_exit(void *retval);
```

- **pthread_exit** 退出线程
- **return**退出线程；但是，主控进程return代表退出进程
- **exit**代表退出整个**进程**





#### 线程回收 join

```
int pthread_join(pthread_t thread, void **retval);
```

- thread：创建的时候传出的第一个参数
- retval：传出线程的退出信息

```C
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
void * thr(void* arg){
    //sleep(5);
	return (VOid*)100;
}

int main(){
	pthread_t tid;
	pthread_create(&tid,NULL,thr,NULL);
    
	void *ret;
    pthread_join(tid,&ret);//线程回收
    printf("ret exit with %d\n",(int)ret);
    
	pthread_exit(NULL);
}
```

阻塞等待回收



#### 杀死线程 cancel

```
int pthread_cancel(pthread_t thread);
```

pthread_cancel====kill

- 输入线程id：tid
- 返回：失败errno，成功0

```C
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
void * thr(void* arg){
    while(1){
    	printf("I am a thread,very happy! tid=%lu\n",pthread_self());
    	sleep(1);
    }
	return NULL;
}

int main(){
	pthread_t tid;
	pthread_create(&tid,NULL,thr,NULL);
    
    sleep(5);
    pthread_cancel(tid);//杀死进程
	void *ret;
    pthread_join(tid,&ret);//线程回收
    printf("ret exit with %d\n",(int)ret);
  
	return 0;
}
```





#### 线程分离 detach

```
int pthread_detach(pthread_t thread);
```

此时不需要pthread_join回收资源

```C
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
void * thr(void *arg){
	printf("I am a thread,tid=%lu\n",pthread_self());
    sleep(4);
    printf("I am a thread,tid=%lu\n",pthread_self());
    return NULL;
}
int main(){
    pthread_t tid;
    pthread_create(&tid,NULL,thr,NULL);
    
    pthread_detach(tid);//线程分离,后面不需join了
    
    sleep(5);
    int ret=0;
    if(pthread_join(tid,NULL)>0){
        printf("join err:%d,%s\n",ret,strerror(ret));
    }
    
    return 0;
}
```



#### 判断线程ID是否相等

线程id在进程内部是唯一的，在外部不一定唯一

```
int pthread_equal(pthread_t t1, pthread_t t2);
```





#### 线程属性设置分离

```C
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);//初始化线程属性
int pthread_attr_destroy(pthread_attr_t *attr);//销毁线程属性
```



```C
//设置属性分离态
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
```

attr：init初始化的属性

detachstate：

PTHREAD_CREATE_DETACHED 线程分离
PTHREAD_CREATE_JOINABLE 允许回收



#### 线程使用注意事项

查看线程库版本 

```C
getconf GUN_LIBPTHREAD_VERSION
```

创建多少个线程？cpu核数*2+2



注意：

1. 主线程退出，其他线程不退出，主线程应调用pthread_exit

2. 避免僵尸线程：

3. ```
   pthread_join
   pthread_detach
   pthread_create 指定分离属性
   ```

4. malloc和mmap申请的内存可以被其他线程释放

5. 应避免在多线程模型中调用fork，除非马上exec

6. 避免在多线程中使用信号机制

















