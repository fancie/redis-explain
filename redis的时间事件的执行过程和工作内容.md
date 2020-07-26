以下内容基于redis3.0
----------------------------------------------
redis的事件事件和文件事件都是定义在“aeEventLoop”结构体中，执行的时候也统一通过“aeProcessEvents”函数执行。
```c
typedef struct aeEventLoop {
    ......
    // 用于生成时间事件 id
    long long timeEventNextId;

    // 最后一次执行时间事件的时间
    time_t lastTime;     /* Used to detect system clock skew */
    ........
    // 时间事件 （只有一个类型，就是跑serverCron函数）
    aeTimeEvent *timeEventHead;
    .......
} aeEventLoop;
```
首先redis在启动的时候初始化时间事件，“aeCreateTimeEvent”函数比较简单，目的是把“aeTimeEvent”初始化到“aeEventLoop”中，并制定执行函数“serverCron”。
```c
// 为 serverCron() 创建时间事件
    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        redisPanic("Can't create the serverCron time event.");
        exit(1);
    }
```
初始化好之后就是前面所说的不断的执行“aeProcessEvents”函数，“aeProcessEvents”函数前面部分都是与时间有关的计算，这些内容都是与时间事件相关的。
```c
int processed = 0, numevents;

/* Nothing to do? return ASAP */
/* !2 = !1 = 0 */
if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

/* Note that we want call select() even if there are no
 * file events to process as long as we want to process time
 * events, in order to sleep until the next time event is ready
 * to fire. */
if (eventLoop->maxfd != -1 ||
    ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
    int j;
    aeTimeEvent *shortest = NULL;
    struct timeval tv, *tvp;

    // 获取最近的时间事件
    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
        shortest = aeSearchNearestTimer(eventLoop);
    if (shortest) {
        // 如果时间事件存在的话
        // 那么根据最近可执行时间事件和现在时间的时间差来决定文件事件的阻塞时间
        long now_sec, now_ms;

        /* Calculate the time missing for the nearest
         * timer to fire. */
        // 计算距今最近的时间事件还要多久才能达到
        // 并将该时间距保存在 tv 结构中
        aeGetTime(&now_sec, &now_ms);
        tvp = &tv;
        tvp->tv_sec = shortest->when_sec - now_sec;
        if (shortest->when_ms < now_ms) {   //毫秒数不够，需要向进一位
            tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
            tvp->tv_sec --;
        } else {
            tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
        }

        // 时间差小于 0 ，说明事件已经可以执行了，将秒和毫秒设为 0 （不阻塞）
        if (tvp->tv_sec < 0) tvp->tv_sec = 0;
        if (tvp->tv_usec < 0) tvp->tv_usec = 0;
    } else {

        // 执行到这一步，说明没有时间事件
        // 那么根据 AE_DONT_WAIT 是否设置来决定是否阻塞，以及阻塞的时间长度

        /* If we have to check for events but need to return
         * ASAP because of AE_DONT_WAIT we need to set the timeout
         * to zero */
        if (flags & AE_DONT_WAIT) {
            // 设置文件事件不阻塞
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        } else {
            /* Otherwise we can block */
            // 文件事件可以阻塞直到有事件到达为止
            tvp = NULL; /* wait forever */
        }
    }

    // 处理事件，阻塞时间由 tvp 决定
    // 系统调用最多超时tvp（里面有纳秒转换），如果在tvp事件内有事件进来（比如get请求），这时会立即开始执行get请求，
    // 但是执行完get请求之后进入下一次循环，这时距离上一次的事件事件超时事件需要重新计算
    numevents = aeApiPoll(eventLoop, tvp);
        ...............
```
代码看着很复杂的样子，实际上就是计算一个时间给“aeApiPoll”函数用，应用超时等待，比如配置的频率上1秒一次，上次用系统获得新的文件事件只消耗了0.5秒，那此次超时时间最多就是0.5秒了。

接下来进入时间事件的处理函数：
```c
/* Process time events
 *
 * 处理所有已到达的时间事件
 */
static int processTimeEvents(aeEventLoop *eventLoop) {
    ..........
    // 遍历链表
    // 执行那些已经到达的事件
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        ............
        // 执行事件处理器，并获取返回值
        retval = te->timeProc(eventLoop, id, te->clientData);
        ............
        // 因为执行事件之后，事件列表可能已经被改变了
        // 因此需要将 te 放回表头，继续开始执行事件
        te = eventLoop->timeEventHead;
        ............
    }
    return processed;
}
```
我们看到时间事件处理函数首先从“eventLoop”拿取时间事件的表头，然后while循环里面的所有时间事件，核心代码就是执行“timeProc”函数，本来初始化的时候只有一个时间事件，然后通过执行完之后挪动表头继续执行，如果时间吻合就不断执行，我们可以打个端点尝试一下，如果执行时间超过配置的时间事件配置的频率执行时间的话，本次while循环会不断重复执行。

时间事件具体执行内容
----------------
接下我们重点来看“timeProc”函数执行的内容，也就是初始化指定的“serverCron”函数。“serverCron”函数体很大，做了不少事情，目前我也还没看完，尴尬！！！！

首先做一些常规操作，记录时钟，记录服务器的内存峰值，是否收到关闭信号等。
```c
server.lruclock = getLRUClock();

/* Record the max memory used since the server was started. */
// 记录服务器的内存峰值
if (zmalloc_used_memory() > server.stat_peak_memory)
    server.stat_peak_memory = zmalloc_used_memory();

/* Sample the RSS here since this is a relatively slow call. */
server.resident_set_size = zmalloc_get_rss();

/* We received a SIGTERM, shutting down here in a safe way, as it is
 * not ok doing so inside the signal handler. */
// 服务器进程收到 SIGTERM 信号，关闭服务器
if (server.shutdown_asap) {

    // 尝试关闭服务器
    if (prepareForShutdown(0) == REDIS_OK) exit(0);

    // 如果关闭失败，那么打印 LOG ，并移除关闭标识
    redisLog(REDIS_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");
    server.shutdown_asap = 0;
}
```
执行一定的次数打印数据库信息，我们看看宏定义“run_with_period”，server.hz是可以配置的，比如系统默认是10，这时打印信息将5秒执行一次（这也看日志级别，默认是不输出”REDIS_VERBOSE“级别的）。
```c
#define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))

// 打印数据库的键值对信息
run_with_period(5000) {
    for (j = 0; j < server.dbnum; j++) {
        long long size, used, vkeys;

        // 可用键值对的数量
        size = dictSlots(server.db[j].dict);
        // 已用键值对的数量
        used = dictSize(server.db[j].dict);
        // 带有过期时间的键值对数量
        vkeys = dictSize(server.db[j].expires);

        // 用 LOG 打印数量
        if (used || vkeys) {
            redisLog(REDIS_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
            /* dictPrintStats(server.dict); */
        }
    }
}
```
如果不是“SENTINEL”模式就打印客户端连接信息，“SENTINEL”模式资料请读者自行搜索。
```c
if (!server.sentinel_mode) {
      run_with_period(5000) {
          redisLog(REDIS_VERBOSE,
              "%lu clients connected (%lu slaves), %zu bytes in use",
              listLength(server.clients)-listLength(server.slaves),
              listLength(server.slaves),
              zmalloc_used_memory());
      }
  }
```
接下来检查客户端，对数据库进行各种操作。
```c
/* We need to do a few operations on clients asynchronously. */
  // 检查客户端，关闭超时客户端，并释放客户端多余的缓冲区
  clientsCron();

  /* Handle background operations on Redis databases. */
  // 对数据库执行各种操作
  databasesCron();
```
接下来就是处理“BGSAVE”相关内容，“BGSAVE”是渐进式的，“SAVE”的时候redis其他工作是不执行的，为了不影响正常的用户操作，redis也支持“BGSAVE”，就是后台跑一个子进程，代码量太大，就不贴代码了。

接下来是AOF，集群，主从复制相关的内容，这个读者自己看吧。
```c
// 根据 AOF 政策，
// 考虑是否需要将 AOF 缓冲区中的内容写入到 AOF 文件中
/* AOF postponed flush: Try at every cron cycle if the slow fsync
 * completed. */
if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

/* AOF write errors: in this case we have a buffer to flush as well and
 * clear the AOF error in case of success to make the DB writable again,
 * however to try every second is enough in case of 'hz' is set to
 * an higher frequency. */
run_with_period(1000) {
    if (server.aof_last_write_status == REDIS_ERR)
        flushAppendOnlyFile(0);
}

/* Close clients that need to be closed asynchronous */
// 关闭那些需要异步关闭的客户端
freeClientsInAsyncFreeQueue();

/* Clear the paused clients flag if needed. */
clientsArePaused(); /* Don't check return value, just use the side effect. */

/* Replication cron function -- used to reconnect to master and
 * to detect transfer failures. */
// 复制函数
// 重连接主服务器、向主服务器发送 ACK 、判断数据发送失败情况、断开本服务器超时的从服务器，等等
run_with_period(1000) replicationCron();

/* Run the Redis Cluster cron. */
// 如果服务器运行在集群模式下，那么执行集群操作
run_with_period(100) {
    if (server.cluster_enabled) clusterCron();
}

/* Run the Sentinel timer if we are in sentinel mode. */
// 如果服务器运行在 sentinel 模式下，那么执行 SENTINEL 的主函数
run_with_period(100) {
    if (server.sentinel_mode) sentinelTimer();
}

/* Cleanup expired MIGRATE cached sockets. */
// 集群。。。TODO
run_with_period(1000) {
    migrateCloseTimedoutSockets();
}
```
-------------------------------------------------------------
2020年7月26日整理于杭州

--fancie
