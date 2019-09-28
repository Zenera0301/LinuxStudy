# Linux系统编程-多线程同步

线程访问同一个共享资源，需要协调步骤！

解决同步问题：加锁



首先安装manpages：manpages-posix-dev

Ubuntu：`sudo apt-get install manpages-posix-dev`





#### 1 互斥量mutex的使用（重要）

两个线程访问同一块共享资源，如果不协调顺序，容易造成数据混乱；

##### 互斥量使用步骤：

- 初始化
- 加锁
- 执行逻辑--操作共享数据
- 解锁

> 注意事项：加锁需要最小粒度，不要一直占用临界区！

##### 函数清单：

- pthread_mutex_init 初始化锁

- pthread_mutex_lock 加锁

- pthread_mutex_unlock 解锁

- pthread_mutex_destroy 摧毁锁

  

mutex互斥量初始化：

```C
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```



给共享资源加锁：

```C
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);//初始化的锁；若未锁，加锁；若已锁，阻塞等待！
int pthread_mutex_trylock(pthread_mutex_t *mutex);//返回值>0说明报错
int pthread_mutex_unlock(pthread_mutex_t *mutex);//解锁
```





摧毁锁：

```C
#include <pthread.h>

int pthread_mutex_destroy(pthread_mutex_t *mutex);
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
const pthread_mutexattr_t *restrict attr);
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```



加锁示例代码：

```C
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
#include<stdlib.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int sum=0;

void * thr1 (void * arg){
        while(1){
                //先上锁
                pthread_mutex_lock(&mutex);

            	//临界区
                printf("hello");
                sleep(rand()%3);
                printf("world\n");
            
                //释放锁
                pthread_mutex_unlock(&mutex);

                sleep(rand()%3);
        }
}
void *thr2(void*arg){
        while(1){
                //先上锁
                pthread_mutex_lock(&mutex);

                printf("HELLO");
                sleep(rand()%3);
                printf("WORLD\n");
               //释放锁
                pthread_mutex_unlock(&mutex);

                sleep(rand()%3);
        }
}

int main(){
        pthread_t tid[2];
        pthread_create(&tid[0],NULL,thr1,NULL);
        pthread_create(&tid[1],NULL,thr2,NULL);
        pthread_join(tid[0],NULL);

        pthread_join(tid[1],NULL);
        return 0;
}
```



运行和结果：

```shell
dj@dj:~/dingjing/mutex$ vi pthread_print.c 
dj@dj:~/dingjing/mutex$ gcc pthread_print.c -lpthread
dj@dj:~/dingjing/mutex$ ./a.out
helloworld
HELLOWORLD
helloworld
HELLOWORLD
helloworld
HELLOWORLD
helloworld
HELLOWORLD
helloworld
helloworld
HELLOWORLD
helloworld
^C
dj@dj:~/dingjing/mutex$ 
```



#### 2 什么叫死锁，以及解决方案

访问共享资源的时候，**互斥量只是建议锁**

死锁分为两种情况：

1. 锁了又锁，自己加了一次锁成功，后来又加了一次锁。第二次锁必然阻塞

   解决办法：加锁时候写简单点。或trylock。

   

2. 交叉锁 访问一块共享资源的时候需要两把锁，其中一个线程占了第一把锁，另一个线程占了第二把锁，两个线程互相申请对方的锁的时候，互相阻塞了

   解决办法：

   每个线程申请锁的顺序要一致（规定先申请key1，再申请key2）；

   如果申请到一把锁，申请另外一般的时候申请失败，应该释放已经掌握的（更讲究）。



#### 3 读写锁的使用

读写锁特点：读共享，写独占，写优先级高

读写锁仍然是一把锁，有不同的状态：

- 未加锁
- 读锁
- 写锁

场景：

- 线程A加写锁成功，线程B请求读锁：B堵塞
- 线程A持有读锁，线程B请求写锁：B阻塞
- 线程A持有读锁，线程B请求读锁：B加锁成功
- 线程A持有读锁，线程B请求写锁，线程C请求读锁：BC阻塞；A释放后B加锁；B释放后C加锁
- 线程A持有写锁，线程B请求读锁，线程C请求写锁：BC阻塞；A释放后C加锁；C释放后B加锁



读写锁使用场景适合**读**的线程多



锁的创建和销毁：

```C
#include <pthread.h>

int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
const pthread_rwlockattr_t *restrict attr);
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
```



读锁：

```C
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);//
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);//try一下，尝试加锁，不会阻塞
```



写锁：

```C
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
```



未加锁：

```C
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock); 
```



```C
#include<stdio.h>
#include<unistd.h>
#include<pthread>

pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;//初始化
int beginnum = 1000;

void * thr_write(void *arg){
    while(1){
        pthread_rwlock_wrlock(&rwlock);
        printf("-%s--self--%lu--beginnum=%d\n",__FUNCTION__,pthread_self,++beginnum);
        usleep(2000);//模拟占用时间
        pthread_rwlock_wrlock(&rwlock);
        usleep(4000);
    }
    return NULL;
}

void * thr_read(void *arg){
    while(1){
        pthread_rwlock_rdlock(&rwlock);
        printf("--%s--self--%lu--beginnum=%d\n",__FUNCTION__,pthread_self,beginnum);
        usleep(2000);//模拟占用时间
        pthread_rwlock_rdlock(&rwlock);
        usleep(2000);
    }
    return NULL;
}

int main(){
    int n=8;i=0;
    pthread_t tid[8];
    for(i=0;i<5;i++){
        pthread_create(&tid[i],NULL,thr_read,NULL);
    }
    for(;i<8;i++){
        pthread_create(&tid[i],NULL,thr_read,NULL);
    }
    for(i=0;i<8;i++){
        pthread_join(tid[i],NULL);
    }
}
```



#### 4 条件变量的使用（重要）

条件变量 可以引起阻塞，并非锁

竞争者可以阻塞在**框里是否有饼**这个条件上

烙饼的人往框里放饼，发出消息有饼了，竞争者开始竞争

生产者和消费者模型：生产者负责生产，消费者负责消费，生产者生产了后，通知消费者，消费者竞争，胜者消费，资源消费完后，消费者阻塞等待。

##### 条件变量作用：避免没有必要的竞争

条件变量

```C
#include <pthread.h>
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
       pthread_mutex_t *restrict mutex,
       const struct timespec *restrict abstime);
int pthread_cond_wait(pthread_cond_t *restrict cond,
       pthread_mutex_t *restrict mutex);
```
条件变量不是锁，要和互斥量组合使用

首先加锁了，timedwait**先释放锁 mutex，再阻塞在条件变量cond上**，实现多个线程阻塞到同一个条件变量上。



销毁一个条件变量；初始化一个条件变量

```C
int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_init(pthread_cond_t *restrict cond,
           const pthread_condattr_t *restrict attr);
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```

唤醒至少一个阻塞在条件变量cond上的线程

```C
int pthread_cond_signal(pthread_cond_t *cond);
```



唤醒阻塞在条件变量cond上的全部线程

```C
int pthread_cond_broadcast(pthread_cond_t *cond);
```



#### 5 条件变量实现的生产消费者模型



#### 6 信号量实现的生产消费者模型

#### 信号量的概念和函数

信号量，加强版的互斥锁



```C
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
```



```C
#include <semaphore.h>

int sem_wait(sem_t *sem);

int sem_trywait(sem_t *sem);

int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);
```





```C
#include <semaphore.h>

int sem_destroy(sem_t *sem);
```

