### 端口复用最常用的用途是: 

- 防止服务器重启时之前绑定的端口还未释放 
- 程序突然退出而系统没有释放端口



### 设置方法: 

```C
int opt = 1;  

setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, (const void *)&opt, sizeof(opt));  
```



> 注意事项: 绑定之前设置端口复用的属性！

