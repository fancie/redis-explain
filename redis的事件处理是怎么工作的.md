以下内容基于redis3.0
----------------------------------------------

redis的事件处理是通过一个永不停歇的函数处理，具体如下：
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
