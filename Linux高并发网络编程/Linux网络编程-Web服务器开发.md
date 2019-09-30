Linux网络编程-web服务器开发



服务器端伪代码：

```c
void main(){
    //修改进程的工作目录（path从命令行传入）
    chdir(path);
    //创建监听套接字
    int lfd = socket(af_inet,sock_stream,0);
    //绑定
    struct sockaddr_in serv;
    serv.family = af_inet;
    serv.port = htons(8989);
    //监听
    listen();
    //阻塞等待请求
    int cfd = accept();
    //读数据
    read(cfd,buf,sizeof(buf));
    //将buf（请求行，请求头，空行，数据）中的请求行拿出来  GET /hello.c http/1.1
    char method[12],path[1024],protocol[12];
    //得到完整的文件名（跳过斜线）
    char * file = path + 1;
    //打开文件
    int fdd = open(file,O_RFONLY);
    int len = 0;
    //把响应头发送出去
    http_respond_head(cfd,"text/plain");
    //循环读数据
    while((len=read(fdd,buf,sizeof(buf)))>0){
        //数据发送给浏览器（套接字通信）
        write(fdd,buf,len);
    }
    
}
//服务器回复给客户端的相应消息
void http_respond_head(int cfd, char* type){
    char buf[1024];
    //状态行
    sprintf(buf,"http/1.1 200 OK\r\n");
    write(cfd,buf,strlen(buf));
    
    //消息报头
    sprintf(buf,"Content-Type:%s\r\n",tpye);
    write(cfd,buf,strlen(buf));
       
    //空行
    write(cfd,"\r\n",2);
}
```

