# Linux的基础语法

### 1 vi编辑器

- 命令行模式
- 末行模式 
- 编辑模式

编辑器操作：
【[n]x】删除光标位置后面n个字符
【[n]X】删除光标位置前面n个字符
【D】删除光标所在位置后面到行尾的所有字符
【[n]dd】删除光标所在行及下面n行   剪切
【p】在光标下一行粘贴
【[n]yy】复制光标所在行及下面n行 
【dG】删除光标所在行到文件结尾
【J】合并光标所在行和下一行 中间用空格连接
【.】执行上一次命令行操作
【u】撤销

编辑器定位：
【ctrl+b】回滚 行号减小
【ctrl+f】前滚 行号增加
【gg】定位在文件第一行行首
【G】定位在文件最后一行行首
【/.】查找任意一个字符
【/*】查找任意多个字符

编辑器替换：
【r】替换光标所在位置的字符
【:r 文件名】在光标当前行的下一行插入一个文件 每次添加只能添加一个文件
【:s/a/b/g】将光标所在行的a替换为b
【:g/a/s//b/g】将文件中所有a替换为b
【:n1,n2s/a/b/g】将行区间n1到n2的行中所有的a替换为b

编辑器设置：
【:set ic】搜索时不区分大小写
【:set noic】搜索时区分大小写



### 2 GCC编译器

#### gcc 整个编译过程

hello.c

gcc -E hello.c 预处理，头文件展开，宏替换 得到 hello.i

gcc -S hello.i 生成汇编代码 hello.s

gcc -c hello.s 将汇编编译成二进制文件 hello.o

ld 链接，得到可执行文件 a.out



#### gcc参数

直接通过`gcc hello.c` 也能直接生成`a.out`文件

`-I ./include/`包含头文件路径

`-o app.out` 指定编译出来的可执行文件的名字，运行是 ./app.out

`-L` 包含库路径

`-l`指定库名 libxxx.a|so , -lxxx

`-g` 加上gdb调试信息

`-Wall` 显示更多的警告

`-lstdc++`编译c++代码

`-O1`或 `-O2`或 `-O3`  优化代码



`gcc hello.c -I./include/ -D DEBUG -o app -g -Wall -O1`



#### g++编译

`g++ hello.cpp` == `gcc hello.cpp -lstdc++ -o a.out`   最后都能`./a.out`





### 3 静态库

linux下静态库以.a结尾 -------windows下静态库以.lib结尾

编译的时候编译成源码，存放在内存中的代码段



优点：执行快；发布应用时不需要发布库

缺点：执行程序体积会变大；库变更时需要重新编译应用



使用：编译时需要加静态库名（含路径），-I包含头文件（大写的i），-L包含库路径，-l指定库名（库名是libCalc时，写作-lCalc:`gcc main.c -o app -I include/ -L lib/ -lCalc `



**制作静态库步骤**：

1. 生成.o  `gcc -c *.c -I ./include`

2. 打包.o文件 `ar rcs libxxx.a *.o`

3. 将头文件与库文件一起发布，移动到lib文件夹下：`mv libCalc.a ../lib/`

   

### 4 动态库(共享库)，首选

linux下共享库以.so结尾



优点：执行程序体积小，当很多应用程序都要用同个库的时候优势明显；库变更时，一般不需要重新编译应用

缺点：执行时需要加载动态库，相对而言，比静态库慢；发布应用时需要同时发布动态库



**制作动态库步骤**：

1. 生成.o，关键参数-fPIC（与位置无关）：`gcc -fPIC -c *.c -I ../include/`
2. 打包 -shared,-o指定库名，后面加上原材料*.o：`gcc -shared -o libCalc.so *.o`
3. 发布：将动态库移动到lib文件夹下：`mv libCalc.so ../lib/`



**使用动态库的方法**（略麻烦）：

使用编译的时候与静态库一样，需要配置库的路径，保证ld链接器能够加载

1. 拷贝或链接到/lib,/usr/lib这样的系统库目录
2. 设置环境变量，LD_LIBRAY_PATH，export LD_LIBRAY_PATH=libpath:$LD_LIBRAY_PATH
3. 修改/etc/ld.so.conf,Idconfig 这两步都需要root权限



法1：拷贝到系统库目录下，不允许！

`gcc main.c -o newapp -I include/ -L lib/ -lCalc`生成可执行文件newapp

`./newapp`运行时报错

查看可执行文件的链接：`ldd newapp` 提示libCalc.so是无法找到

将我们的libCalc.so文件的软链接放到系统能找到的库路径，**源文件要用绝对路径**：

`sudo ln -s /home/dingjing/study/Calc/lib/libCalc.so /lib/libCalc.sp`

再次查看可执行文件的链接：`ldd newapp`提示libCalc.so找到了





法2：修改LD_LIBRAY_PATH环境变量，将库所在的路径，添加到环境变量中，用冒号分隔，不推荐！

`export LD_LIBRAY_PATH = /home/dingjing/study/Calc/lib/ : $LD_LIBRAY_PATH`



**法3：修改/etc下的配置文件ld.so.conf，将库路径添加到该文件中，推荐！！**

`sudo vi /etc/ld.so.conf` 把/home/dingjing/study/Calc/lib/写入文件的新的一行

`sudo ldconfig -v`加-v显示加载过程（是小写的L）





### 5 makefile

优点：一次编写，终身受益

命名规则：makefile、Makefile



三要素：

目标：希望生成的目标文件

依赖：通过什么文件去生成

规则命令：怎么去生成



写法：

目标:依赖

tab键规则命令

第一版：

```shell
app:main.c add.c sub.c div.c mul.c

​	gcc -o app -I ./include main.c add.c sub.c div.c mul.c
```

如果更改其中一个文件，所有源码都重新编译

可以考虑编译过程分解，现生成.o文件，使用.o文件生成结果。这样，如果一个.c文件发生变化，只需要把变化的文件生成.o文件



第二版：

```shell
app:main.o add.o sub.o div.o mul.o
	gcc -o app -I ./include main.o add.o sub.o div.o mul.o
main.o:main.c
	gcc -c main.c -I ./include
add.o:add.c
	gcc -c add.c -I ./inlcude
sub.o:sub.c
	gcc -c sub.c -I ./include
mul.o:mul.c
	gcc -c mul.c -I ./inlcude
div.o:div.c
	gcc -c div.c -I ./include
```



第三版：

```shell
#新建变量ObjFiles 定义目标文件
ObjFiles=main.o add.o sub.o div.o mul.o
#目标文件用法$(var) 直接翻译 直译
app:$(ObjFiles)
	gcc -o app -I ./include main.o add.o sub.o div.o mul.o
main.o:main.c
	gcc -c main.c -I ./include
add.o:add.c
	gcc -c add.c -I ./inlcude
sub.o:sub.c
	gcc -c sub.c -I ./include
mul.o:mul.c
	gcc -c mul.c -I ./inlcude
div.o:div.c
	gcc -c div.c -I ./include
```

隐含规则：默认处理第一个目标



第四版：

两个函数：

- wildcard文件匹配
- patsubst内容替换

自动变量：只能在规则中出现

-  `$@ `代表目标
- `  $^ `  代表全部依赖
- ` $< ` 代表第一个依赖
- ` $?`代表第一个变化的依赖

```shell
#get all .c files
SrcFiles=$(wildcard *.c)

#all .c files-->.o files
ObjFiles=$(patsubst %.c,%.o,$(SrcFiles))

all:app app1

#目标文件用法 $(var) 直接翻译 直译
app1:$(ObjFiles)
	gcc -o $@ -I ./include $(SrcFiles)
app2:$(ObjFiles)
	gcc -o $@ -I ./include $(SrcFiles)
	
#模式匹配规则
%.o:%.c
	gcc -o $@ -c $< -I ./include
	
test:
	@echo $(SrcFiles)
	@echo $(ObjFiles)

#定义伪目标，防止有歧义
.PHONY:clean all

clean:
	@rm -f *.o #或者 -@rm *.o
	@rm -f app1 app2
```

@在规则前代表不输出该条规则命令

规则前的减号 - 代表该条规则报错，仍然继续执行

`make -f makefile1`指定makefile文件





### 6 gdb调试

使用gdb：编译的时候加-g参数

启动 `gdb app`      



在gdb里启动程序： 

`r 或者 run`：运行     

`start`：停留在main函数，分步调试，敲n，表示next，执行下一条指令，之后回车就行

`s 或者 step`：分步执行，进入到函数内部执行，但进不去库函数



`quit`：退出gdb调试



#### 设置运行参数

 `set args 521 518 `：设置启动参数 或`run 521 518`

`set argv[1]="12"`



#### 显示10行代码

`l 或 list`：默认显示10行代码，回车，再显示接下来的10行代码；`list main.c:1`从第一行开始显示



#### 断点

`b 17`或者 ` break 17`：给17行设置断点，run的时候直接到断点处

`b main.c:17`:给main.c文件的第17行设置断点

`b main` ：给main函数设置断点

`info b`：查看设置了哪些断点

`d 4` 和 `del 6`：删除断点

`c`代表continue，程序继续执行到下一个断点

`b 32 if i==1`:**条件断点**，当i==1的时候，进行这个断点



#### 查看变量

`p argc`  和 `print i`都可以查看变量的值，用法相同

`ptype i` 查看变量的类型



#### 跟踪变量

`display argc`:跟踪变量的值，每执行一次命令，显示跟踪的变量的值

`info display`:查看跟踪的变量编号

`undisplay 1`:放弃跟踪1号变量



#### 调试core文件（gdb跟踪core）

core----案发现场

`ulimit -c unlimited`：设置生成core，大小可以无限大

`./app`:报错：段错误（核心已转储）

`ls -lrt`:出现一个core文件

`gdb app core`:告诉你程序是哪行出错了

`where`：再次告诉你是哪行出错了



设置core文件格式：

`sudo vi /proc/sys/kernel/core_pattern`

`core-%e-%t`

或：

`sudo echo "core-%e-%t">/proc/sys/kernel/core_pattern`





系统API和库函数的关系



### 7 文件IO操作

#### open

`man 2 open`

```C++
   #include <sys/types.h>
   #include <sys/stat.h>
   #include <fcntl.h>

   int open(const char *pathname, int flags);
   int open(const char *pathname, int flags, mode_t mode);
```
pathname：文件名

flags：

```c++
//必选项：
O_RDONLY:只读
O_WRONLY：只写
O_RDWR：读写

//可选项：
追加O_APPEND 
创建O_CREAT:mode权限位(mode & ~umask) 
非阻塞O_NONBLOCK
```

返回值：open返回最小的可用文件描述符；失败返回-1



#### close

```
int close(int fd);
```

fd：open打开的文件描述符

返回值：成功返回0，失败返回-1，设置errno



#### read

```
ssize_t read(int fd,void*buf,size_t count);
```

fd:文件描述符

buf:缓冲区

count:缓冲区大小

返回值：失败返回-1，设置errno；成功返回读到的大小；0代表读到文件末尾



#### write

```
ssize_t write(int fd,const void *buf,size_t count);
```

fd:文件描述符

buf:缓冲区

count:缓冲区大小

返回值：失败返回-1，设置errno；成功返回写入的字节数；0代表未写入





#### open/close 可以实现touch功能

```
#include<stdio.h>
#include<sys/types.h>
#include<sys/fcntl.h>
#include<unistd.h>

int main(int argc,char *argv[])
{
	if(argc!=2){

		printf("./a.out filename\n");
		return -1;
	 }

	int fd = open(argv[1],O_RDONLY|O_CREAT,0666);
	close(fd);
	return 0;
}
```



#### read/write 查看文件内容，可以实现cat功能

```C++
#include<stdio.h>
#include<unistd.h>
#include<sys/stat.h>
#include<sys/types.h>
#include<fcntl.h>

int main(int argc,char *argv[]){
	if(argc!=2){
		printf("./a.out filename\n");
		return -1;
	}
	int fd = open(argv[1],O_RDONLY);
	//读 输出到屏幕
	char buf[256];
	int ret = read(fd,buf,sizeof(buf));
	write(STDOUT_FILENO,buf,ret);
	close(fd);
	return 0;
}
```



#### lseek

```
off_t lseek(int fd,off_t offset,int whence);
```

fd:文件描述符

offset:偏移量

whence: SEEK_SET文件开始的位置；SEEK_CUR当前位置；SEEK_END结尾

返回值：成功，返回当前位置到开始的长度；失败返回-1，设置errno



打开一个文件，输入一段内容，然后读取一下该文件的内容，输出到屏幕

lseek 改变读写位置，计算大小，拓展文件（至少写一次）



#### 阻塞fcntl

read函数在读设备、管道、网络的时候

输入输出设备对应/dev/tty

```
int fcntl(int fd,int cmd,...)
```

fcntl决策函数，设置非阻塞，就3步，3行代码

```
int flags = fcntl(fd,F_GETFL);//获取原来的样子
flags |= O_NONBLOCK;//与非阻塞或一下
fcntl(fd,F_SETFL,flags); //设置一下
```



#### errno



#### dup2

主要用于文件重定向

```
int dup2(int oldfd,int newfd);//关闭新的fd，并将新的fd指向旧的fd
```



#### dup

用于复制，便于恢复现场





### 8 文件操作函数

stat:获取inode信息

access：按有效用户ID和有效组ID测试，追述符号链接（可读、可写、可执行、测试文件是否存在）

chmod：修改文件权限

chown：修改用户 及 组

truncate：截断文件长度为指定长度

rename：重命名

link: 

link：硬链接

symlink：软链接

readlink：读取符号链接本身内容，得到链接指向的文件名

unlink：删除符号链接  或  硬链接计数





getcwd：获得当前目录

chdir：改变进程工作目录

mkdir：创建目录

rmdir：删除目录

opendir：打开目录

readdir：读目录

rewinddir：把目录指针恢复到起始位置

telldir：获取目录读写位置

seekdir：修改目录读写位置

closedir：关闭目录

