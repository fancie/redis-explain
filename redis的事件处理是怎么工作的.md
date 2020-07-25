以下内容基于redis3.0
----------------------------------------------

redis的事件处理是通过一个永不停歇的函数处理（除非进程收到stop请求），具体如下：
```c
/*
 * 事件处理器的主循环
 */
void aeMain(aeEventLoop *eventLoop) {

    eventLoop->stop = 0;

    while (!eventLoop->stop) {
        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```
其中最出要的处理函数在这里：
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
```
aeProcessEvents处理所有的事件，包括文件事件和时间事件，文件事件具体指从客户端收到的所有指令，例如get/set等，时间事件是redis内部处理一些必要的操作，例如主动清除过期键，更新统计信息等，具体函数是redis.c/serverCron方法。

本次文档主要介绍文件事件，时间事件请看《redis的时间事件做了一些什么事情》。

首先redis在启动的时候初始化事件，并生成一个内核事件队列：kqueue()。（我用的是mac调试的，使用的是kqueue，实际上redis还提供另外3种方式：select、evport、epoll）。
```c
/*
 * 初始化事件处理器状态
 */
aeEventLoop *aeCreateEventLoop(int setsize) {
..........
if (aeApiCreate(eventLoop) == -1) goto err;

    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    // 初始化监听事件
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
..........
}
```
然后为TCP连接关联应答处理器，for循环里面有2个，分别表示Tcp和Tcp6：
```c
// 为 TCP 连接关联连接应答（accept）处理器
    // 用于接受并应答客户端的 connect() 调用
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                redisPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```
实际上说调用aeApiAddEvent，把EVFILT_READ添加到内核事件队列，核心代码是如下2句（实际上aeApiAddEvent可以添加两种类型的事件，读和写，这时候只需要读，如果有信息要会写给客户端，会执行写相关的代码）：
```c
EV_SET(&ke, fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
if (kevent(state->kqfd, &ke, 1, NULL, 0, NULL) == -1) return -1;
```
到这里的时候与redis的文件事件相关的已经初始化好了，如果没有客户端接入，redis不会执行任何文件事件（但是redis会定时执行时间事件）。

当用户执行“telnet 127.0.0.1 6379”命令的时候，redis会接入一个客户端（具体参考《telnet 6379命令是怎么执行的》），用户接着执行get/set的时候，这时redis开始处理文件事件。

那redis是
