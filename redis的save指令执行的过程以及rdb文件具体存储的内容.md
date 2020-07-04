以下内容基于redis3.0
----------------------------------------------
redis的save指令是把当前内存的内容以一定的格式写入文件，redis可以通过load文件重新读取内容加载到内存，不至于数据丢失，save指令是阻塞的（我们可以
通过测试发现save不执行完，其他指令例如get是无法执行的），相对save的还有bgsave指令，后期我也将介绍bgsave指令是怎么执行的。

save指令执行的是以下方法：
```c
/* Save the DB on disk. Return REDIS_ERR on error, REDIS_OK on success 
 *
 * 将数据库保存到磁盘上。
 *
 * 保存成功返回 REDIS_OK ，出错/失败返回 REDIS_ERR 。
 */
int rdbSave(char *filename)
```
方法开始做一些常规操作，比如创建文件，初始化IO这样的操作，接着设置校验函数。
```c
// 设置校验和函数
if (server.rdb_checksum)
    rdb.update_cksum = rioGenericUpdateChecksum;
```
校验函数是指所有内容用一个算法计算出一个结果，redis这里会计算出一个8个字节的校验和，这里有点像MD5，把整个文件内容生成一个MD5值，文章底部有校验函数相关内容。

写入版本号，这里用REDIS0006表示（redis3.0是0006）

接下来是循环进入每个redis数据库（redis默认创建了16个数据库，我们在使用redis的时候如果不指定选择哪个数据库，系统采用0号数据库保存内容）。

具体怎么写入数据库，我们后面介绍，我们先略过这个部分，看看无数据的情况还做了哪些事情。

接下来写入结束标记
```c
if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;
```
再写入8个字节的校验和
```c
cksum = rdb.cksum;
memrev64ifbe(&cksum);
rioWrite(&rdb,&cksum,8);
```

最后冲洗缓存，保存文件，改名等。

我们来看看没有任何内容的rdb文件是什么样的。

当我们执行如下命令：od -c dump.rdb，获得下面这些内容：
```c
0000000    R   E   D   I   S   0   0   0   6 377   ܳ  **   C 360   Z 362
0000020  362   V                                                        
0000022
```

前面9个字节代表rdb文件的redis版本号：redis0006

接下来的“337”，代表数据库的结束标记

后面8个字节是校验和，校验和是读取的时候用于计算数据库内容有没有被修改或者数据内容是否完整。

我们另外看一下有数据的情况，当我们执行“set key value”之后rdb内容是什么样的。

继续执行od -c dump.rdb指令，获得如下内容：
```c
0000000    R   E   D   I   S   0   0   0   6 376  \0  \0 003   k   e   y
0000020  005   v   a   l   u   e 377     322   W   A 376 205 237 176    
0000037
```
这里跟空数据库不同的是从376开始到value结束的部分，这其中：

376：代表选择数据库，由如下执行执行：
```c
if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
```
“\0”：表示选择了0号数据库

“\0 003 k e y 005 v a l u e”，这个我们分解来看

  “\0”：表示数据类型，这是表示value的类型为“REDIS_RDB_TYPE_STRING”类型，它与内存数据的类型“REDIS_STRING”相对应。
  
  “003”：表示接下来的（k，v）的key需要读取3个字节。
  
  “k e y”：表示（k，v）的k的具体值。
  
  “005”：表示“REDIS_RDB_TYPE_STRING”类型的value需要读取5个字节，这里表示为字符串，也就是“value”。
  
  “value”：就是（k，v）的v的具体值。

############华丽的分界线############################################################

下面我们来看看有数据的情况下是怎么写入的。

首先是遍历所有redis的数据库，遍历到某个数据库的时候获得dict的迭代器。
```c
// 指向数据库
redisDb *db = server.db+j;
// 指向数据库键空间
dict *d = db->dict;
// 跳过空数据库
if (dictSize(d) == 0) continue;
// 创建键空间迭代器
di = dictGetSafeIterator(d);
```
然后通过迭代器遍历所有的dictEntry，把key和value写入到文件。
```c
//这里的sds是char *的一个扩转，兼容C语言原始的char *，同时记录字节长度，扩转了很多方法，例如字符串拼接。
sds keystr = dictGetKey(de);    
//robj是redis定义的结构体，兼容redis使用的各种类型，例如REDIS_STRING、REDIS_LIST、REDIS_SET、REDIS_ZSET、REDIS_HASH。
robj key, *o = dictGetVal(de);  
long long expire;
// 根据 keystr ，在栈中创建一个 key 对象
initStaticStringObject(key,keystr);
// 获取键的过期时间
expire = getExpire(db,&key);
// 保存键值对数据
if (rdbSaveKeyValuePair(&rdb,&key,o,expire,now) == -1) goto werr;
```
save指令不保存过期的数据，所以先检查过期时间

具体保存key，value的部分在“rdbSaveKeyValuePair”方法里面，接下来我们看“rdbSaveKeyValuePair”这个方法。

核心代码在这里：
```c
/* Save type, key, value 
*
* 保存类型，键，值
*/
if (rdbSaveObjectType(rdb,val) == -1) return -1;
if (rdbSaveStringObject(rdb,key) == -1) return -1;
if (rdbSaveObject(rdb,val) == -1) return -1;
```
从代码我们可以看到首先保存value的类型，再保存string类型的key，最后保存value对象。

rdbSaveObjectType方法比较简单，就是od指令里面“003”之前的“\0”，每个数字代表一种数据类型。

本次我们只解析string类型的内容如何保存到rdb中，所以保存key和value使用的是同样的方法。
```c
/* Save a string object as [len][data] on disk. If the object is a string
 * representation of an integer value we try to save it in a special form 
 *
 * 以 [len][data] 的形式将字符串对象写入到 rdb 中。
 *
 * 如果对象是字符串表示的整数值，那么程序尝试以特殊的形式来保存它。
 *
 * 函数返回保存字符串所需的空间字节数。
 */
int rdbSaveRawString(rio *rdb, unsigned char *s, size_t len){
    //方法体
}
```
正如注释中写的string内容的数据是以 【len】【data】 的格式将字符串对象写入到 rdb 中。

这跟我们用od命令观察的结果是一样的，比如：003 k e y 005 v a l u e。

首先看数据是否可以进行整数编码，整数编码可以更节省空间，例如：”set k1 125“这样的数据，value部分如果保存为字符串需要3个字节“1 2 5”，当系统对数据进行整数编码之后只有1个字节。
```c
/* Try integer encoding 
 *
 * 尝试进行整数值编码
 */
if (len <= 11) {
    unsigned char buf[5];
    if ((enclen = rdbTryIntegerEncoding((char*)s,len,buf)) > 0) {
        // 整数转换成功，写入
        if (rdbWriteRaw(rdb,buf,enclen) == -1) return -1;
        // 返回字节数
        return enclen;
    }
}
```
如果不能进行整数编码，然后会尝试对数据进行压缩，压缩的算法可以参考：<<第一篇：redis的lzf压缩和解压算法解析>>，如果压缩成功，将会保存一个压缩标记和压缩之前和之后的长度，具体实现：
```c
// 写入类型，说明这是一个 LZF 压缩字符串
byte = (REDIS_RDB_ENCVAL<<6)|REDIS_RDB_ENC_LZF;
if ((n = rdbWriteRaw(rdb,&byte,1)) == -1) goto writeerr;
nwritten += n;

// 写入字符串压缩后的长度
if ((n = rdbSaveLen(rdb,comprlen)) == -1) goto writeerr;
nwritten += n;

// 写入字符串未压缩时的长度
if ((n = rdbSaveLen(rdb,len)) == -1) goto writeerr;
nwritten += n;

// 写入压缩后的字符串
if ((n = rdbWriteRaw(rdb,out,comprlen)) == -1) goto writeerr;
nwritten += n;
```
如果数据既不能整数编码也不能被压缩，那直接写入长度和内容。
```c
// 写入长度
if ((n = rdbSaveLen(rdb,len)) == -1) return -1;
nwritten += n;

// 写入内容
if (len > 0) {
    if (rdbWriteRaw(rdb,s,len) == -1) return -1;
    nwritten += len;
}
```
到这里的时候基本上string类型的（k，v）所需要的type、key、value都已经写入。

redis支持多种数据结构，除了REDIS_STRING之外还有：REDIS_LIST，REDIS_SET、REDIS_ZSET、REDIS_HASH。

其他数据的写入有些额外的处理，但是基本上最终都是调用基本数据类型写入的，例如字符串、int、double类型的格式写入的，这里就不分解描述了。

附加说明（校验函数）
----------------
如果启用校验功能，校验函数是在每次调用“rioWrite”的时候计算。
```c
static inline size_t rioWrite(rio *r, const void *buf, size_t len){...}
```
校验函数本身也比较简单，就是对传入上一次结果、本次数据的内容、本次数据的字节长度，然后循环进行查表操作、位移操作、按位异或操作，大家可以看看代码，crc64_tab实际上就是一个数组，里面包含256个8字节长度的数据。
```c
uint64_t crc64(uint64_t crc, const unsigned char *s, uint64_t l) {
    uint64_t j;

    for (j = 0; j < l; j++) {
        uint8_t byte = s[j];
        crc = crc64_tab[(uint8_t)crc ^ byte] ^ (crc >> 8);
    }
    return crc;
}
```

-------------------------------------------------------------
2020年7月4日整理于杭州

--fancie
