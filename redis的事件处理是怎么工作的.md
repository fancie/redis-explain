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

首先redis在启动的时候初始化事件，redis初始化的时候维护了一个10000+的事件队列，并生成一个内核事件队列：kqueue()。（我用的是mac调试的，使用的是kqueue，实际上redis还提供另外3种方式：select、evport、epoll）。
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

当用户执行“telnet 127.0.0.1 6379”命令的时候，redis会接入一个客户端,这时redis开始处理文件事件（具体参考《telnet 6379命令是怎么执行的》），用户接着执行get/set的时候也类似。

那redis如何监测文件事件的呢？主要是通过如下函数完成：
```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    if (tvp != NULL) {
        //超时限制，如果有新的事件，比如read事件，读取完内容直接返回，如果什么事件都没有，就阻塞到超时事件结束
        struct timespec timeout;
        timeout.tv_sec = tvp->tv_sec;
        timeout.tv_nsec = tvp->tv_usec * 1000;
        retval = kevent(state->kqfd, NULL, 0, state->events, eventLoop->setsize,
                        &timeout);
    } else {
        retval = kevent(state->kqfd, NULL, 0, state->events, eventLoop->setsize,
                        NULL);
    }

    if (retval > 0) {
        int j;
        
        numevents = retval;
        for(j = 0; j < numevents; j++) {
            int mask = 0;
            struct kevent *e = state->events+j;
            
            if (e->filter == EVFILT_READ) mask |= AE_READABLE;
            if (e->filter == EVFILT_WRITE) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->ident; 
            eventLoop->fired[j].mask = mask;           
        }
    }
    return numevents;
}
```
这涉及到内核的多路复用，内核维护一个事件处理队列，如果队列中的事件有新的事件要处理，则通过kevent返回给redis进程，redis根据内核的事件fd判断是哪个客户端的消息，然后把结果返回给相应的客户端，所以aeApiPoll核心代码主要是kevent(state->kqfd, NULL, 0, state->events, eventLoop->setsize,&timeout)，如果有待处理的事件，就把这些事件放置到eventLoop->fired中，接下来通过循环eventLoop->fired分别处理相应的请求。

为了便于连接多路复用和事件队列，我从网上找了一个客户端和服务器通信的案例，实际上redis跟这个类似，案例包括server和client两个部分。

server：
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/event.h>
#include <sys/socket.h>

#include <sys/time.h>
#include <sys/types.h>

#include <unistd.h>
#include <fcntl.h>




#define BACKLOG 5           //完成三次握手但没有accept的队列长度
#define CONCURRENT_MAX 8    //应用层同时可以处理的连接
#define SERVER_PORT 9588
#define BUFFER_SIZE 1024
#define QUIT_CMD "quit"

int clients[CONCURRENT_MAX];



void check(int fd, const char *errorMsg, const char *sucMsg) {
    if (fd < 0) {
        perror(errorMsg);
        exit(1);
    } else {
        printf("%s", sucMsg);
    }
}


int main(int argc, char *argv[]) {
    char input_msg[BUFFER_SIZE];
    char recv_msg[BUFFER_SIZE];



    /* struct sockaddr_in {
     *  __uint8_t       sin_len;
     *  sa_family       sin_family;     //协议族，AF_xx
     *  in_port_t       sin_port;       //端口号，网络字节顺序
     *  struct in_addr  sin_addr;       //ip地址，4字节
     *  char            sin_zero[8];    //8字节，为了和sockaddr保持一样大小
     * }
     * int 为机器字长 2/4字节
     * short int 2字节
     * long int 4字节
     * long long 8字节
     */
    struct sockaddr_in addr4;
    memset(&addr4, 0, sizeof(addr4));
    addr4.sin_len = sizeof(struct sockaddr_in);
    addr4.sin_family = AF_INET;
    addr4.sin_addr.s_addr = inet_addr("127.0.0.1");
    addr4.sin_port = htons(SERVER_PORT);


    int server = socket(AF_INET, SOCK_STREAM, 0);
    check(server, "server create failed \n", "create good \n");

    int b = bind(server, (struct sockaddr *) &addr4, sizeof(struct sockaddr));
    check(b, "server bind failed \n", "bind good \n");

    int l = listen(server, BACKLOG);
    check(l, "server listen failed \n", "listen good \n");


    struct timespec timeout = {10, 0};

    int kq = kqueue();
    check(kq, "create queue failed \n", "kqueue good \n");


    struct kevent changes;
    EV_SET(&changes, STDIN_FILENO, EVFILT_READ, EV_ADD, 0, 0, NULL);
    kevent(kq, &changes, 1, NULL, 0, NULL);

    EV_SET(&changes, server, EVFILT_READ, EV_ADD, 0, 0, NULL);
    kevent(kq, &changes, 1, NULL, 0, NULL);

    struct kevent events[CONCURRENT_MAX + 2];

    while (1) {
        int ret = kevent(kq, NULL, 0, events, 10, &timeout);
        if (ret < 0) {
            printf("kevent error \n");
            continue;

        } else if (ret == 0) {
            printf("kevent timeout \n");
            continue;

        } else {
            printf("kevent good \n");

            for (int i = 0; i < ret; i++) {
                struct kevent current_event = events[i];

                //server input
                if (current_event.ident == STDIN_FILENO) {
                    bzero(input_msg, BUFFER_SIZE);
                    fgets(input_msg, BUFFER_SIZE, stdin);

                    //quit
                    if (strncmp(input_msg, "quit", 4) == 0) {
                        exit(0);
                    }

                    //broadcast
                    for (int i = 0; i < CONCURRENT_MAX; i++) {
                        if (clients[i] != 0) {
                            send(clients[i], input_msg, BUFFER_SIZE, 0);
                        }
                    }

                    //new connect
                } else if (current_event.ident == server) {
                    struct sockaddr_in client_addr;

                    socklen_t addr_len;

                    int client_fd = accept(server, (struct sockaddr *) &client_addr, &addr_len);

                    if (client_fd > 0) {
                        int index = -1;

                        //put in clients array
                        for (int i = 0; i < CONCURRENT_MAX; i++) {
                            if (clients[i] == 0) {
                                index = i;
                                clients[i] = client_fd;
                                break;
                            }
                        }

                        if (index >= 0) {
                            //监听该客户端输入
                            EV_SET(&changes, client_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
                            //放入事件队列
                            kevent(kq, &changes, 1, NULL, 0, NULL);

                            printf("新客户端（fd=%d）加入成功 %s:%d \n", client_fd, inet_ntoa(client_addr.sin_addr),
                                   ntohs(client_addr.sin_port));
                        } else {
                            bzero(input_msg, BUFFER_SIZE);
                            strcpy(input_msg, "服务器加入的客户端数已达最大值，无法加入\n");
                            send(client_fd, input_msg, BUFFER_SIZE, 0);
                            printf("新客户端加入失败 %s:%d \n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                        }
                    }

                    //client new msg
                } else {


                    bzero(recv_msg, BUFFER_SIZE);
                    long byte_num = recv((int) current_event.ident, recv_msg, BUFFER_SIZE, 0);
                    if (byte_num > 0) {
                        if (byte_num > BUFFER_SIZE) {
                            byte_num = BUFFER_SIZE;
                        }
                        recv_msg[byte_num] = '\0';
                        printf("客户端(fd = %d):%s\n", (int) current_event.ident, recv_msg);
                        char *replay = "我收到了";
                        if (send(current_event.ident, replay, BUFFER_SIZE, 0) == -1) {
                            perror("发送消息出错!\n");
                        }
                    } else if (byte_num < 0) {
                        printf("从客户端(fd = %d)接受消息出错.\n", (int) current_event.ident);
                    } else {
                        EV_SET(&changes, current_event.ident, EVFILT_READ, EV_DELETE, 0, 0, NULL);
                        kevent(kq, &changes, 1, NULL, 0, NULL);
                        close((int) current_event.ident);
                        for (int i = 0; i < CONCURRENT_MAX; i++) {
                            if (clients[i] == (int) current_event.ident) {
                                clients[i] = 0;
                                break;
                            }
                        }
                        printf("客户端(fd = %d)退出了\n", (int) current_event.ident);
                    }
                }
            }
        }
    }
}
```
client:
```c
#include <stdio.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

#define BUFFER_SIZE 1024

int main (int argc, const char * argv[]) {
    struct sockaddr_in server_addr;
    server_addr.sin_len = sizeof(struct sockaddr_in);
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(9588);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    bzero(&(server_addr.sin_zero), 8);

    int server_sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock_fd == -1) {
        perror("socket error");
        return 1;
    }
    char recv_msg[BUFFER_SIZE];
    char input_msg[BUFFER_SIZE];

    if (connect(server_sock_fd, (struct sockaddr *) &server_addr, sizeof(struct sockaddr_in)) == 0) {
        fd_set client_fd_set;
        struct timeval tv;
        tv.tv_sec = 20;
        tv.tv_usec = 0;


        while (1) {
            FD_ZERO(&client_fd_set);
            FD_SET(STDIN_FILENO, &client_fd_set);
            FD_SET(server_sock_fd, &client_fd_set);

            int ret = select(server_sock_fd + 1, &client_fd_set, NULL, NULL, &tv);
            if (ret < 0) {
                printf("select 出错!\n");
                continue;
            } else if (ret == 0) {
                printf("select 超时!\n");
                continue;
            } else {
                if (FD_ISSET(STDIN_FILENO, &client_fd_set)) {
                    bzero(input_msg, BUFFER_SIZE);
                    fgets(input_msg, BUFFER_SIZE, stdin);
                    if (send(server_sock_fd, input_msg, BUFFER_SIZE, 0) == -1) {
                        perror("发送消息出错!\n");
                    }
                }

                if (FD_ISSET(server_sock_fd, &client_fd_set)) {
                    bzero(recv_msg, BUFFER_SIZE);
                    long byte_num = recv(server_sock_fd, recv_msg, BUFFER_SIZE, 0);
                    if (byte_num > 0) {
                        if (byte_num > BUFFER_SIZE) {
                            byte_num = BUFFER_SIZE;
                        }
                        recv_msg[byte_num] = '\0';
                        printf("服务器:%s\n", recv_msg);
                    } else if (byte_num < 0) {
                        printf("接受消息出错!\n");
                    } else {
                        printf("服务器端退出!\n");
                        exit(0);
                    }

                }
            }
        }

    }
}
```
-------------------------------------------------------------
2020年7月25日整理于杭州

--fancie
