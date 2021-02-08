# Nginx 的reload，热升级以及优雅的关闭流程

[toc]

## 一、reload流程

1. 向master进程发送HUP信号（reload命令）
2. master进程校验配置语法是否正确；
3. master打开可能引入的新的监听端口；
4. master用新的配置文件启动新的worker子进程；
5. 启动新的worker子进程之后，master向老的worker子进程发送QUIT信号（优雅的退出）；
6. 老的子进程收到QUIT信号之后，关闭监听句柄（也就是说，新的连接只会到新的子进程），处理完当前的连接后就结束进程；



## 二、热升级的流程

1. 将久的Nginx文件替换成新的Nginx文件（注意备份）；

2. 向master进程发送USR2信号；

   > Nginx还没有提供相应的命令行

3. 之后，现有的master进程会修改pid文件名，加后缀oldbin；

   > 这是为了给新的master进程让路；

4. 老的master进程，用新的Nginx二进制文件，启动新的master进程；

   > 此时，会出现两个master进程和老的worker进程。新的master进程会启动新的worker进程

5. 向老的master进程发送QUIT信号，关闭老的进程；

   > 此时老的master进程会关闭老的worker进程。此时，热升级已经结束，老的master进程会一直保留，这是为了方便回滚。

6. 回滚：向老的master进程发送HUP，向新的master发送QUIT；

## 三、优雅的关闭

优雅的关闭，是对worker进程而言的。因为只有worker进程才会处理请求。

优雅的关闭，是指worker进程可以识别当前的连接没有处理请求，这个时候再将连接关闭。

对于有一个请求，nginx是做不到的。比如：用nginx代理websocket协议的时候；对于TCP协议的时候，nginx做不到。

**优雅的关闭主要是针对STTP协议的。**

1. 设置一个定时器

   `worker_shutdown_timeout`，nginx会设置一个标示为，表示进入优雅的关闭的流程。

2. 关闭监听句柄

   保证worker进程不会处理新的连接了。

3. 关闭所有的空闲连接

4. 在循环中等待全部连接关闭

   如果等待的时候超过了设定的`worker_shutdown_timeout`时间，连接也会被强制关闭。

5. 退出进程

