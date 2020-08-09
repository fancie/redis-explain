以下内容基于redis3.0
----------------------------------------------
和Mysql主从复制的原因一样，Redis虽然读取写入的速度都特别快，但是也会产生读压力特别大的情况。为了分担读压力，Redis支持主从复制，Redis的主从结构可以采用一主多从或者级联结构，Redis主从复制可以根据是否是全量分为全量同步和增量同步。

redis开启主从复制比较简单，我用clion启动3个redis实例，每个实例指定不同的配置文件，在clion中就是创建多个“CMake Application”，然后在“Program arguments”栏中指定相应的配置文件即可，具体配置也简单，只要修改几个配置就行，具体如下。
```c
我使用3个实例，分别是：
Master：127.0.0.1:6379
Slave: 127.0.0.1:6380  和  127.0.0.1:6381

Master/redis.conf配置修改内容：
bind 127.0.0.1
port 6379

Slave/slave1.conf配置修改内容：
dbfilename dump1.rdb
port 6380
slaveof 127.0.0.1 6379

Slave/slave2.conf配置修改内容：
dbfilename dump2.rdb
port 6381
slaveof 127.0.0.1 6379
```
这样配置完之后，即可通过clion启动3个实例，可以从Console日志中查看redis主从是否成功。

如果发现Master报错如下（相应的slave也有错误）：

这是因为第一次全量同步的时候是使用rdb文件同步的，可以在Master里面执行“save”命令生成rdb文件，这样重新启动slave的时候就不会报错了。
```c
SYNC failed. Can't open/stat DB after BGSAVE: No such file or directory
```
同步过程
---------
1.首先Slave通过“replicationCron”函数轮询连接Master，实际上调用这个函数：
```c
// 以非阻塞方式连接主服务器
int connectWithMaster(void) {
  redisLog(3, "调用connectWithMaster函数");
  int fd;

  // 连接主服务器
  fd = anetTcpNonBlockConnect(NULL,server.masterhost,server.masterport);
  ........
}
```
2.这时Master收到一个读取文件事件（Slave发送过来的连接请求），建立连接。

3.Slave通过调用“syncWithMaster”函数发送“PING”指令，Master回复”PONG“指令。

4.Slave通过“slaveTryPartialResynchronization”函数发送“PSYNC”指令，Master回复”FULLRESYNC“开始同步。

5.服务发送rdb内容给客户端，Slave通过“readSyncBulkPayload”函数读取内容。

6.如果文件过大，Master会分多次发送给Slave，通过“CONTINUE”指令识别。

我们看看服务器打印日志：
```c
Slave日志：
[1572] 09 Aug 23:39:51.520 * Connecting to MASTER 127.0.0.1:6379
[1572] 09 Aug 23:39:51.520 # 调用connectWithMaster函数
[1572] 09 Aug 23:39:53.242 * MASTER <-> SLAVE sync started
[1572] 09 Aug 23:39:54.834 # 调用syncWithMaster函数
[1572] 09 Aug 23:39:54.834 * Non blocking connect for SYNC fired the event.
[1572] 09 Aug 23:39:54.835 # 调用syncWithMaster函数
[1572] 09 Aug 23:39:54.835 * Master replied to PING, replication can continue...
[1572] 09 Aug 23:39:54.835 # 调用sendSynchronousCommand函数
[1572] 09 Aug 23:39:54.836 # 调用slaveTryPartialResynchronization函数
[1572] 09 Aug 23:39:54.836 * Partial resynchronization not possible (no cached master)
[1572] 09 Aug 23:39:54.836 # 调用sendSynchronousCommand函数
[1572] 09 Aug 23:39:54.837 * Full resync from master: 7590edc09d354b09697b2231ed4a525ab18a5af0:2269
[1572] 09 Aug 23:39:54.837 # 调用replicationDiscardCachedMaster函数
[1572] 09 Aug 23:39:55.593 # 调用readSyncBulkPayload函数
[1572] 09 Aug 23:39:55.593 * MASTER <-> SLAVE sync: receiving 18 bytes from master
[1572] 09 Aug 23:39:55.593 # 调用readSyncBulkPayload函数
[1572] 09 Aug 23:39:55.594 * MASTER <-> SLAVE sync: Flushing old data
[1572] 09 Aug 23:39:55.594 * MASTER <-> SLAVE sync: Finished with success

Master日志：
[1148] 09 Aug 23:39:54.836 # 调用syncCommand函数
[1148] 09 Aug 23:39:54.836 * Slave asks for synchronization
[1148] 09 Aug 23:39:54.836 # 调用masterTryPartialResynchronization函数
[1148] 09 Aug 23:39:54.836 * Full resync requested by slave.
[1148] 09 Aug 23:39:54.836 * Starting BGSAVE for SYNC
[1148] 09 Aug 23:39:54.837 * Background saving started by pid 1576
[1148] 09 Aug 23:39:54.837 # 调用replicationScriptCacheFlush函数
[1576] 09 Aug 23:39:54.844 * DB saved on disk
[1148] 09 Aug 23:39:55.593 * Background saving terminated with success
[1148] 09 Aug 23:39:55.593 # 调用updateSlavesWaitingBgsave函数
[1148] 09 Aug 23:39:55.593 # 调用replicationCron函数
[1148] 09 Aug 23:39:55.593 # 调用sendBulkToSlave函数
[1148] 09 Aug 23:39:55.593 * Synchronization with slave succeeded
```

内容写得过于简单，下次有时间写详细一点。

-------------------------------------------------------------
2020年8月09日整理于杭州

--fancie
