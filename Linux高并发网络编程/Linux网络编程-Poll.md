### Poll

底层是链表

#### 优缺点

优点： poll支持的文件描述符数量可以突破1024

缺点：

- 每次调用select，**都需要把fd集合从用户态拷贝到内核态**，这个开销在fd很多时会很大 
- 同时每次调用select，**都需要在内核遍历传递进来的所有fd**，这个开销在fd很多时也很大 
- 不支持跨平台



### Select

底层是数组，一旦分配写死了，大小就固定了

#### 优缺点

优点： 跨平台

缺点：

- 每次调用select，**都需要把fd集合从用户态拷贝到内核态**，这个开销在fd很多时会很大 
- 同时每次调用select，**都需要在内核遍历传递进来的所有fd**，这个开销在fd很多时也很大 
- select**支持的文件描述符数量太小了**，默认是**1024**



### epoll

底层通过红黑树实现

使用共享内存，来回切换的时候不需要拷贝





select突破不了1024限制（文件描述符个数上限1024个），poll和epoll可以突破；

ulimit -a 可以查看资源上限，open file 是1024，需要修改配置文件，重新编译内核才能实现：

`sudo vi /etc/security/limits.conf`

