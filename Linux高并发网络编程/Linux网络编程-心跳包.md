# Linux网络编程-心跳包



作用：判断客户端和服务器是否处于连接状态



1. 心跳机制 

   不会携带大量的数据

   每隔一定时间服务器->客户端/客户端->服务器发送一个数据包 

   

2. 心跳包看成一个协议 ：应用层协议 

   

3. 判断网络是否断开 

   有多个连续的心跳包没收到/没有回复 

   关闭通信的套接字 

   

4. 重连 

   重新初始套接字 

   继续发送心跳包 



