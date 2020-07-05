以下内容基于redis3.0
----------------------------------------------

redis的通用hash函数是把传入的char * 和长度，通过一定的计算转化成32位无符号的整数。
```c
unsigned int dictGenHashFunction(const void *key, int len) {...}
```
首先定义种子和一个固定的“m”值，种子是启动的时候赋值当前时间的int值，“m”值在redis里面是一个定值。

然后初始化hash结果变量”h“，h = seed ^ len；
```c
uint32_t seed = dict_hash_function_seed;
const uint32_t m = 0x5bd1e995;
const int r = 24; //用于获取int的第一个字节

/* Initialize the hash to a 'random' value */
uint32_t h = seed ^ len;
```
通过不断的乘法、位移、异或计算，把结果“h”压缩到4个字节，程序每次取四个字节，如果key的长度小于4个字节，这部分将不执行。
```c
const unsigned char *data = (const unsigned char *)key;
    while(len >= 4) {
        uint32_t k = *(uint32_t*)data;

        k *= m;
        k ^= k >> r;
        k *= m;

        h *= m;
        h ^= k;

        data += 4;
        len -= 4;
    }
```
处理最后的3个字节，为什么是3个字节？前面部分因为每次处理都是取4个字节，所以后面最多剩余3个字节未处理。
```c
switch(len) {
    case 3:
      h ^= data[2] << 16;
    case 2:
      h ^= data[1] << 8;
    case 1:
      h ^= data[0];
      h *= m;
  };
```
做一些最后的操作，以确保最后几个字节很好的合并到了一起，因为前面4个字节的时候都是按照4字节计算的，最后3个字节是按照每个字节的数据分开计算的。
```c
h ^= h >> 13;
h *= m;
h ^= h >> 15;
```
最后就是把h返回，“h”是一个无符号的整形，而且很多操作是异或，所以数据比较大，不会出现特别小的数值，实际在使用过程中会根据需要压缩到相应的的长度，比如新的dict初始化新的table的时候默认都是比较小的，在使用过程中再扩容。

我们看看实际使用过程中，怎么用“h”匹配dict里面table的数量，“h”与sizemask做一次“&”运算。
```c
int idx = h & d->ht[table].sizemask;
```
redis的hash函数就讲到这里吧，下次有时间我们测试一下效果，比如输入一组连续的值，看看是不是均匀分布到结果空间。

-------------------------------------------------------------
2020年7月5日整理于杭州

--fancie
