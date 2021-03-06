以下内容基于redis3.0
----------------------------------------------

redis所有事件处理都是在这里：
```c
ae.c文件的：int aeProcessEvents(aeEventLoop *eventLoop, int flags)
```
常用的get，set等都是在这里处理的，telnet也不例外

aeProcessEvents的前面部分是检查是否有事件正在处理需要阻塞，例如save命令，config配置的hz频率等。

接下来获取事件的fd（已就绪文件描述符）
```c
// 处理文件事件，阻塞时间由 tvp 决定
// 阻塞是让serverCron执行完毕，如果serverCron还未执行完就阻塞
// 为什么会阻塞？serverCron固定保持server.hz节奏执行，可能执行的事件when_sec会大于now_sec
numevents = aeApiPoll(eventLoop, tvp);
```

ae_kqueue使用kqueue，kqueue是unix下的一个IO多路复用库。因为我是在mac上运行，所以执行：
```c
ae_kqueue.c的static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp)
```
aeApiPoll获得是TCP连接还是TCP6连接。

然后获得具体的文件事件：aeFileEvent
```c
// 从已就绪数组中获取事件
aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
```
eventLoop->events  里面有2个文件事件，这是redis启动的时候注册的，具体在：
```c
redis.c的listenToPort(server.port,server.ipfd,&server.ipfd_count){....}
```
listenToPort注册2个文件事件，一个是TCP，另外一个是TCP6
```c
fds[*count] = anetTcp6Server(server.neterr,port,NULL,
    server.tcp_backlog);
if (fds[*count] != ANET_ERR) {
    anetNonBlock(NULL,fds[*count]);
    (*count)++;
}
fds[*count] = anetTcpServer(server.neterr,port,NULL,
    server.tcp_backlog);
if (fds[*count] != ANET_ERR) {
    anetNonBlock(NULL,fds[*count]);
    (*count)++;
}
```
文件事件有2个事件处理器，一个用于读，一个用于写。
```c
// 读事件处理器
aeFileProc *rfileProc;
// 写事件处理器
aeFileProc *wfileProc;
```
telnet的rfileProc执行：
```c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask)
```
acceptTcpHandler方法的最后创建一个redisClient，后续命令例如get，set命令都是由redisClient负责处理，每一个新的连接都创建一个redisClient。

telnet命令不需要执行wfileProc事件。

至此telnet命令执行完（后面的是与时间事件记录相关）

-------------------------------------------------------------
2020年7月4日整理于杭州

--fancie
