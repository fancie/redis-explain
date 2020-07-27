以下内容基于redis3.0
----------------------------------------------
Redis除了RDB持久化功能以外，Redis还提供了AOF(Append Only File)持久化功能。与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis所执行的写命令来记录数据库状态的。

redis需要通过配置文件开启，在调试的时候没有传入配置文件可以用代码开启，例如：
```c
server.aof_state = REDIS_AOF_ON;
server.aof_fsync = AOF_FSYNC_EVERYSEC;
```
redis开启需要一个开关和同步方式，我测试的时候采用“AOF_FSYNC_EVERYSEC”，每秒追加一次，redis开启AOF之后，redis在initServer()的时候会打开AOF文件，默认文件名为：appendonly.aof，代码在initServer()中如下：
```c
if (server.aof_state == REDIS_AOF_ON) {
    server.aof_fd = open(server.aof_filename,
                           O_WRONLY|O_APPEND|O_CREAT,0644);
    if (server.aof_fd == -1) {
        redisLog(REDIS_WARNING, "Can't open the append-only file: %s",
            strerror(errno));
        exit(1);
    }
}
```
打开AOF之后，redis启动的时候将从AOF中读取数据初始化redis数据，具体在“loadDataFromDisk(void)”方法里面，redis选择优先读取aof文件。
```c
void loadDataFromDisk(void) {
  // 记录开始时间
  long long start = ustime();

  // AOF 持久化已打开？
  if (server.aof_state == REDIS_AOF_ON) {
      // 尝试载入 AOF 文件
      if (loadAppendOnlyFile(server.aof_filename) == REDIS_OK)
          // 打印载入信息，并计算载入耗时长度
          redisLog(REDIS_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
  // AOF 持久化未打开
  } else {
      // 尝试载入 RDB 文件
      if (rdbLoad(server.rdb_filename) == REDIS_OK) {
          // 打印载入信息，并计算载入耗时长度
          redisLog(REDIS_NOTICE,"DB loaded from disk: %.3f seconds",
              (float)(ustime()-start)/1000000);
      } else if (errno != ENOENT) {
          redisLog(REDIS_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
          exit(1);
      }
  }
}
```
”loadAppendOnlyFile(char *filename)“函数负责读取aof文件里面的数据，因为涉及到数据格式的问题，我们先看写入，最后再来看读取的函数。

讲到这里的时候，redis接下来的启动与aof无关了，接下来我们看写入的情况。

写入
---

当我们敲入“set key value”的时候，按照AOF的同步逻辑，数据应该按照每秒一次的形式写入aof文件，实际上这个过程是在“call”函数中完成的，“call”函数是redis的核心函数，负责执行客户端的命令，call函数有如下代码，负责把命令广播到AOF或者slave中。
```c
// 将命令复制到 AOF 和 slave 节点
if (flags & REDIS_CALL_PROPAGATE) {
    int flags = REDIS_PROPAGATE_NONE;

    // 强制 REPL 传播
    if (c->flags & REDIS_FORCE_REPL) flags |= REDIS_PROPAGATE_REPL;

    // 强制 AOF 传播
    if (c->flags & REDIS_FORCE_AOF) flags |= REDIS_PROPAGATE_AOF;

    // 如果数据库有被修改，那么启用 REPL 和 AOF 传播
    if (dirty)
        flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);

    if (flags != REDIS_PROPAGATE_NONE)
        propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
}

//具体的广播函数
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
               int flags)
{
    // 传播到 AOF
    if (server.aof_state != REDIS_AOF_OFF && flags & REDIS_PROPAGATE_AOF)
        feedAppendOnlyFile(cmd,dbid,argv,argc);

    // 传播到 slave
    if (flags & REDIS_PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc);
}
```
上面代码中的“feedAppendOnlyFile”函数负责把命令输出成一个固定的格式，存放在“aof_buf”中，然后通过“eventLoop->beforesleep(eventLoop)”定期把“aof_buf”中的内容写入到aof文件。

我们可以看到通过“feedAppendOnlyFile”函数，“aof_buf”多了如下内容，“\r\n”我替换成了换行，这样看着清晰。
```c
*2
$6
SELECT
$1
0
*3
$3
set
$3
key
$5
value
```
所以整个过程就是把“set key value”命令替换成了上面一大堆字符，上面字符有两个标记“*”和“$”，“*”代码开启一个新的命令，后面的数字是命令的参数个数，“$”代表开始读取一个参数，后面的数字代表读取的字节长度。

所以：“*2”表示开始一个命令，这个命令有2个参数。

“$3”表示开始读取一个参数，这个参数有3个字节。

所以这里是2个命令的组合：“select 0” 和 “set key value”（redis是需要选择db的，初始化的时候有16个db，默认是第0个，所以是“select 0”）。

最后调用“flushAppendOnlyFile(int force)”写入到aof文件，这个是通过“beforeSleep(struct aeEventLoop *eventLoop)”执行的。
```c
// 每次处理事件之前执行
void beforeSleep(struct aeEventLoop *eventLoop) {
    ..............
    /* Write the AOF buffer on disk */
    // 将 AOF 缓冲区的内容写入到 AOF 文件
    flushAppendOnlyFile(0);
    ..............
}
void flushAppendOnlyFile(int force){
    ........
    nwritten = write(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    ........
}
```
实际上“flushAppendOnlyFile”函数体很大，做了很多事情，例如：写入的出现异常把异常内容移除等，这些内容读者可以自行研究。

读取
---

现在我们回过头来读取的情况，读取是在“loadAppendOnlyFile(char *filename)”函数里面，核心内容如下：
```c
while(1) {
    int argc, j;
    unsigned long len;
    robj **argv;
    char buf[128];
    sds argsds;
    struct redisCommand *cmd;

    /* Serve the clients from time to time 
     *
     * 间隔性地处理客户端发送来的请求
     * 因为服务器正处于载入状态，所以能正常执行的只有 PUBSUB 等模块
     */
    if (!(loops++ % 1000)) {
        loadingProgress(ftello(fp));
        processEventsWhileBlocked();
    }

    // 读入文件内容到缓存
    if (fgets(buf,sizeof(buf),fp) == NULL) {
        if (feof(fp))
            // 文件已经读完，跳出
            break;
        else
            goto readerr;
    }

    // 确认协议格式，比如 *3\r\n
    if (buf[0] != '*') goto fmterr;

    // 取出命令参数，比如 *3\r\n 中的 3
    argc = atoi(buf+1);

    // 至少要有一个参数（被调用的命令）
    if (argc < 1) goto fmterr;

    // 从文本中创建字符串对象：包括命令，以及命令参数
    // 例如 $3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n
    // 将创建三个包含以下内容的字符串对象：
    // SET 、 KEY 、 VALUE
    argv = zmalloc(sizeof(robj*)*argc);
    for (j = 0; j < argc; j++) {
        if (fgets(buf,sizeof(buf),fp) == NULL) goto readerr;

        if (buf[0] != '$') goto fmterr;

        // 读取参数值的长度
        len = strtol(buf+1,NULL,10);
        // 读取参数值
        argsds = sdsnewlen(NULL,len);
        if (len && fread(argsds,len,1,fp) == 0) goto fmterr;
        // 为参数创建对象
        argv[j] = createObject(REDIS_STRING,argsds);

        if (fread(buf,2,1,fp) == 0) goto fmterr; /* discard CRLF */
    }

    /* Command lookup 
     *
     * 查找命令
     */
    cmd = lookupCommand(argv[0]->ptr);
    if (!cmd) {
        redisLog(REDIS_WARNING,"Unknown command '%s' reading the append only file", (char*)argv[0]->ptr);
        exit(1);
    }

    /* Run the command in the context of a fake client 
     *
     * 调用伪客户端，执行命令
     */
    fakeClient->argc = argc;
    fakeClient->argv = argv;
    cmd->proc(fakeClient);

    /* The fake client should not have a reply */
    redisAssert(fakeClient->bufpos == 0 && listLength(fakeClient->reply) == 0);
    /* The fake client should never get blocked */
    redisAssert((fakeClient->flags & REDIS_BLOCKED) == 0);

    /* Clean up. Command code may have changed argv/argc so we use the
     * argv/argc of the client instead of the local variables. 
     *
     * 清理命令和命令参数对象
     */
    for (j = 0; j < fakeClient->argc; j++)
        decrRefCount(fakeClient->argv[j]);
    zfree(fakeClient->argv);
}
```
实际上就是不断的读取aof文件，把里面的内容读取出来组成一个命令，然后用伪客户端执行（redis的所有命令都要通过客户端执行，所以这里生成一个伪客户端，伪客户端不需要回写内容）。

我们来看下面的内容是怎么读取的：
```c
*2
$6
SELECT
$1
0
*3
$3
set
$3
key
$5
value
```
读取第一个命令：

首先读取“*2”，发现这是个命令，这个命令有2个参数，

然后读取“$6”，发现第一个参数有6个字节，接下来读取6个字节，这时候命令变成“SELECT”。

接下来读取第二个参数“$1”，发现这个参数只有一个字节，所以2个参数全部读取完毕，所以命令变成“SELECT 0”，然后执行它。

这时开始读取第二个命令：

首先读取“*3”，发现这是个命令，这个命令有3个参数，

读取第一个参数“$3”，参数长度为3，这个参数是3个字节，命令变成“set”，

读取第二个参数“$3”，参数长度也是3，读取3个字节，命令变成“set key”，

读取第三个参数“$5”，参数长度为5，读取5个字节，命令变成“set key value”，然后执行这个命令。

到这里的时候整个内容读取完毕，执行2个命令“SELECT 0” 和 “set key value”，这与我们开始执行的内容是一致的。

写到这里的时候整个AOF相关的内容都写完了。

-------------------------------------------------------------
2020年7月27日整理于杭州

--fancie
