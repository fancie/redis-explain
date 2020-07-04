以下内容基于redis3.0
----------------------------------------------

redis的get指令大致分为三个步骤（不计算执行之前之前或者执行之后redis的事项以及集群相关的内容）

  第一步：获得指令数据查找具体指令执行的方法并检查指令参数是不是正确以及认证信息，服务器状态等

  第二步：用key查找具体的value，这里分为2个子项：查找是否过期，查找具体的value

  第三步：把结果返回给客户端
  
###################华丽的分界线#######################################

第一步：获得指令数据查找具体指令执行的方法

1）读取客户端的指令内容在networking.c里面
```c
/*
 * 读取客户端的查询缓冲区内容
 */
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask)
2）根据读取的内容查找具体应该执行什么方法
/*
 * 根据给定命令名字（SDS），查找命令
 */
struct redisCommand *lookupCommand(sds name)
```

指令集和相关内容保存在一个hash表中，redis定义的一个结构体：dict
```c
// 命令表
redis.h定义的：dict *commands
dict存放的是：dictEntry
/*
 * 哈希表节点
 */
typedef struct dictEntry {....}
```
再从dictEntry解析出具体的value，就是命令相关内容，这些命令实际上是redis初始化的时候写入dict中，数据来源如下：\<br>
redis.c顶部的：struct redisCommand redisCommandTable[] = {....}\<br>
redisCommandTable记录所有当前redis所能执行的所有指令。
读取指令之后赋值到redisClient的cmd中。
具体怎么通过“get”获得哪个dictEntry在第二个步骤再讲，处理方式跟get指令通过key获取value的方式一致。

接着再检查指令参数是否正确，认证信息，redis服务器状态等，这些都是一些判断，有时间自己阅读。


第二步：用key查找具体的value，这里分为2个子项：查找是否过期，查找具体的value
```c
/* Call() is the core of Redis execution of a command */
// 调用命令的实现函数，执行命令
void call(redisClient *c, int flags)

具体执行的是cmd的proc，proc是在第一步查找之后赋值的
// 执行实现函数
c->cmd->proc(c);
```

接下来分解proc具体干了什么事情，以下方法是读取操作而从数据库中查找返回 key 对于的value值。
```c
// 查找
robj *o = lookupKeyRead(c->db, key)
先检查过期时间
expireIfNeeded(db,key);
再从数据库中取出键的值
val = lookupKey(db,key);
```

实际上2个步骤执行的过程类似，都是用key去一个dict里面查找，具体我们看看lookupKey吧。
lookupKey执行的是dictFind方法
```c
/*
 * 返回字典中包含键 key 的节点
 *
 * 找到返回节点，找不到返回 NULL
 *
 * T = O(1)
 */
dictEntry *dictFind(dict *d, const void *key)
```

每个dict有2张表，如果key存在一般在table[0]能查到，只有在redis需要对table进行整理的时候需要查2张表（例如redis在扩容），如果key在table[0]没有查到，会去查找table[1]。
首先对key进行hash，这跟java语言的map是类似的（本人写java的），dict是用hash+链表存储数据。
具体的计算hash值方法在这里，有兴趣的读者自行研究（多说一句：一个优秀的hash函数需要尽可能均匀分布在某个区间里面，如果一个hash函数计算出来的结果大多相同，这样会导致dict的链表很长，链表需要一个一个查找比对，执行效率比较低）：
```c
unsigned int dictGenHashFunction(const void *key, int len)
```

计算出hash值之后再在dich的table里面看这个位置有没有值，如果有值那这个位置上个链表结构，然后一个个的用key比对：
```c
// 计算索引值
idx = h & d->ht[table].sizemask;
// 遍历给定索引上的链表的所有节点，查找 key
he = d->ht[table].table[idx];
// T = O(1)
while(he) {
    if (dictCompareKeys(d, key, he->key))
        return he;
    he = he->next;
}
```

如果比对成功就代表这个key在dict的table里面，这个dictEntry就是我们需要查找的值。
后面有一些列的操作，比如记录执行多少次指令等，然后把查询到的结果写入缓冲区。

第三步：把结果返回给客户端
```c
// 执行写事件
if (fe->mask & mask & AE_WRITABLE) {
    if (!rfired || fe->wfileProc != fe->rfileProc)
        fe->wfileProc(eventLoop,fd,fe->clientData,mask);
}
wfileProc具体执行的是：
/*
 * 负责传送命令回复的写处理器
 */
void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) 
```

写入内容到套接字，这个跟我们大学时候学的聊天系统客户端与服务器互相发消息类似
```c
// c->sentlen 是用来处理 short write 的
// 当出现 short write ，导致写入未能一次完成时，
// c->buf+c->sentlen 就会偏移到正确（未写入）内容的位置上。
nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen)
```

执行完write之后，我们可以在控制台查看到具体的内容，例如：
```c
set k1 hello
+OK
get k1
$5
hello
```

整个执行过程就写到这里吧。

-------------------------------------------------------------
2020年7月4日整理于杭州
--fancie
