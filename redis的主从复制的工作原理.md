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


-------------------------------------------------------------
2020年7月29日整理于杭州

--fancie
