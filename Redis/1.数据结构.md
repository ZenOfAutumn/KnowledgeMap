# Redis-数据结构

## 简单动态字符串（Simple Dynamic String）

###  源码

```c
struct sdshdr {
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```
###  SDS与C字符串的区别 
1. 返回字符串长度为O(1)而非O(n)。
  如果求C语言中的字符串的长度，直到遇到’\0’结束，时间复杂度为O(n)；而SDS中的字符串长度，有一个成员记录着字符串的长度，所以时间复杂度为O(1)，这确保了获取字符串长度的工作不会成为Redis的性能瓶颈。
2. 杜绝缓冲区溢出 。
  C语言中的strcat()，利用这个函数进行字符串的拷贝时，这个函数内部是默认已经分配够空间，如果我们在调用之前忘记给目的字符串分配够空间，那么就会发生字符串的溢出；而SDS中提供的接口sdscat()，如果空间不够，可以自动扩容。 
3. 减少修改字符串时带来的内存重分配次数 
  - **空间预分配**用于优化SDS的字符串增长的操作；如果SDS的长度小于1MB，那么程序分配和len属性同样大小的未使用空间；如果大于1MB，程序会分配1MB的未使用空间。
  - **惰性空间释放**用于优化SDS的字符串缩短的操作；当程序需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性记录，将回收的空间记录起来。 
4. 二进制安全 
  - C字符串中的字符必须符合某种编码，并且除了字符串的末尾之外，字符串里面不能包含空字符，这个限制使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据；
  - SDS读取数据的时候，是根据len属性进行读取的，所以不存在遇到空字符串就结束的说法，SDS的API都是二进制安全的，所有SDS的API都会以处理二进制的方式处理存在SDS存放在buf数组里的数据，程序不会对其中的数据做任何限制、过滤，数据写入是什么样子，读取出来就是什么样子。把SDS中的buf称为字节数组是因为，它不是用buf来保存字符，而是用来保存一系列二进制数据。Redis不仅可以保存文本数据，还可以保存任意格式的二进制数据。
5. 兼容部分C字符串函数
  虽然SDS的API是二进制安全的，但它们一样遵循C字符串以空字符结尾的惯例，这是为了让那些保存文本数据的SDS可以重用一部分<string.h>库定义的函数。


## 链表（Double Linked List）

### 源码

```c
/*
 * 双端链表节点
 */
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;
```

```c
/*
 * 双端链表结构
 */
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```

### 特性
1. 双链表实现
2. 无环结构
3. 带有表头指针和表尾指针
4. 带有链表长度计数器
5. 节点值用空指针存储，可以多台

## 字典（Hash Map）

### 源码
```c
/*
 * 哈希表
 *
 * 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
 */
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;

/*
 * 哈希表节点
 */
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;

/*
 * 字典
 */
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表 HashTable
    dictht ht[2]; 

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;

/*
 * 字典类型特定函数
 */
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```

### 存储原理
1. 一般情况下，字典只使用ht[0]，只有再哈希的时候才使用ht[1]。
  - 首先对键进行哈希（murmurhash2和DJB HASH）得到哈希值。然后哈希值与哈希表掩码求与得到索引。
  - 采用链地址法解决冲突，但最新插入的节点在链表的头部。
### 再哈希
- Redis 对字典的哈希表执行 rehash 的步骤如下:
1. 为字典的 ht[1] 哈希表分配空间,空间的大小取决于要执行的操作,以及 ht[0]当前包含的键值对数量( used 属性值):
  - 如果执行的是扩展操作,那么ht[1] 的大小为第一个大于等于 ht0].used $\times$ 2的 $2^n$.
  - 如果执行的是收缩操作,那么ht[1]的大小为第一个大于等于 ht[0].used 的$2^n$.
2. 将保存在 ht[0] 中所有键值对 rehash 到 ht[1] 上面:   rehash指的是重新计算键的哈希值和索引值,然后键键值对放到 ht[1] 哈希表的指定位置。
3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后, 释放 ht[0], 再将 ht[1] 设置为 ht[0],并在 ht[1] 后面创建一个空白的哈希表。

- 当哈希表的负载因子达到一定条件时，会进行再哈希以收缩或者扩展：
  - 负载因子计算公式：load_factor=ht[0].used/ht[0].size

### 渐进式再哈希

**再哈希动作并不是一次性,集中式完成的,而是分多次,渐进式完成的。**

- 原因：当哈希表里保存的键值对多至百万甚至亿级别时,一次性地全部 rehash 的话,庞大的计算量会对服务器性能造成严重影响.

- 渐进式 rehash 的步骤:
  1. 为 ht[1] 分配空间。
  2. 在字典中维持一个索引计数器变量 rehashidx， 将它的值设置为0，表示 rehash 正式开始。
  3. 在 rehash 进行期间，每次对字典进行增删改查时，顺带将 ht[0] 在 rehashidx 索引上的所有键值对 rehash 到 ht[1] 中，同时将 rehashidx 加 1。
  4. 随着操作不断进行，最终在某个时间点上,，ht[0] 所有的键值对全部 rehash 到 ht[1] 上，这是讲 rehashidx 属性置为 -1，表示 rehash操作完成。
  5. 在渐进式 rehash 执行期间,新添加到字典的键值对一律保存到 ht[1] 里，不会对 ht[0] 做添加操作,这一措施保证了 ht[0]只减不增，并随着 rehash 进行， 最终变成空表。

渐进式的 rehash 避免了集中式 rehash 带来的庞大计算量和内存操作。

##  跳跃表（Skip List）

### 用途
1. 有序集合ZSET
2. 集群节点用作内部数据结构
### 源码

```C
/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;

/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```
### 属性
1. 每层都有两个属性：前进指针和跨度。前进指针用于从前往后遍历，跨度则用于记录前进节点与当前节点的距离。跨度的作用为在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位。层高为1到32之间的随机数。
2. 后退指针：用于从后往前遍历。每个节点只有一个后退指针，跨度为一。
3. 分值：节点按照所保存的分值从小到大排列。为double浮点数。分值相同的按照成员对象字典序大小排序。
4. 成员对象：保存成员对象。成员对象必须唯一。



## 整数集合（Int Set）

集合只包含整数元素，并且集合元素的数量不多时，使用整数集合。

###  源码
```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```
### 升级
1. contents数组的真正类型取决于encoding属性值，encoding具有三种编码方式，分别为
```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
分别存储16位，32位，64位的整数类型。编码方式取决于最大值所需位数。但只支持因为新增最大值导致的位数升级，而不支持自动降级。
2. contents数组按照从小到大保存集合中的元素。
3. 升级步骤O(N)
  1. 根据新元素类型扩展整数集合底层数组的空间大小，并为新元素分配空间。
  2. 将底层数组现有元素转换成新类型，并按照有序性放入新数组。
  3. 将新元素加入数组。
4. 升级优势
  1. 提升灵活性
  2. 节约内存


##  压缩列表（Zip List）

当列表或哈希只有小整数值和短字符串时，使用压缩列表实现。

### 总体编码
<zlbytes><zltail><zllen><entry><entry><zlend>

- zlbytes：存储一个无符号整数，固定四个字节长度，用于存储压缩列表所占用的字节，当重新分内存的时候使用，不需要遍历整个列表来计算内存大小。

- zltail：存储一个无符号整数，固定四个字节长度，代表指向列表尾部的偏移量，偏移量是指压缩列表的起始位置到指定列表节点的起始位置的距离。

- zllen：压缩列表包含的节点个数，固定两个字节长度，源码中指出当节点个数大于2^16-2个数的时候，该值将无效，此时需要遍历列表来计算列表节点的个数。

- entryX：列表节点区域，长度不定，由列表节点紧挨着组成。

- zlend：一字节长度固定值为255，用于表示列表结束。


### 列表元素编码

<previous_entry_length><encoding><content>

- previous length
  用于存储上一个节点的长度，因此压缩列表可以从尾部向头部遍历，即当前节点位置减去上一个节点的长度即得到上一个节点的起始位置。previous length的长度可能是1个字节或者是5个字节，如上一个节点的长度小于254，则该节点只需要一个字节就可以表示前一个节点的长度了，如果前一个节点的长度大于等于254，则previous length的第一个字节为254，后面用四个字节表示当前节点前一个节点的长度。这么做很有效地减少了内存的浪费。

- encoding
  节点的encoding保存的是节点的content的内容类型以及长度，encoding类型一共有两种，一种字节数组一种是整数，encoding区域长度为1字节、2字节或者5字节长。

- content
  content区域用于保存节点的内容，节点内容类型和长度由encoding决定。

压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。


## 对象

### 类型与编码

一对键值对：键对象和值对象
```c
typedef struct redisObject{
  // 类型
  unsigned type:4;
  // 编码
  unsigned encoding:4;
  // 指向底层实现数据结构的指针
  void *ptr;
} robj;
```

| 对象   | TYPE属性       | 编码                        | 底层数据结构             | 适用条件                       | 转换条件                 | 配置参数                                     |
| ---- | ------------ | ------------------------- | ------------------ | -------------------------- | -------------------- | ---------------------------------------- |
| 字符串  | REDIS_STRING | REDIS_ENCODING_INT        | 整数值                | 可以用long类型表示的整数值            | 变为非整数或超过long范围转换为raw |                                          |
| 字符串  | REDIS_STRING | REDIS_ENCODING_EMBSTR     | 使用embstr编码的简单动态字符串 | 整数或浮点数转换成的字符串及原始字符串小于32个字节 | 只读，一有修改便转换为raw       |                                          |
| 字符串  | REDIS_STRING | REDIS_ENCODING_RAW        | 简单动态字符串            | 整数或浮点数转换成的字符串及原始字符串大于32个字节 |                      |                                          |
| 列表   | REDIS_LIST   | REDIS_ENCODING_ZIPLIST    | 压缩列表               | 元素字符串长度小于64字节且元素数量小于512    | 不满足适用条件转换为linkedlsit | list-max-ziplist-size和list-max-ziplist-entries |
| 列表   | REDIS_LIST   | REDIS_ENCODING_LINKEDLIST | 双端列表               | 不满足以上条件                    |                      |                                          |
| 哈希表  | REDIS_HASH   | REDIS_ENCODING_ZIPLIST    | 压缩列表               | 元素字符串长度小于64字节且元素数量小于512    | 不满足适用条件转换为hashtable  | hash-max-ziplist-entries和hash-max-ziplist-value |
| 哈希表  | REDIS_HASH   | REDIS_ENCODING_HT         | 字典                 | 不满足以上条件                    |                      |                                          |
| 集合   | REDIS_SET    | REDIS_ENCODING_INTSET     | 整数集合               | 元素都是整数且元素数量小于512           | 不满足适用条件转换为hashtable  | set-max-intset-entries                   |
| 集合   | REDIS_SET    | REDIS_ENCODING_HT         | 字典                 | 不满足以上条件                    |                      |                                          |
| 有序集合 | REDIS_ZSET   | REDIS_ENCODING_ZIPLIST    | 压缩列表               | 元素字符串长度小于64字节且元素数量小于128    |                      | zset-max-ziplist-entries 和zset-max-ziplist-value |
| 有序集合 | REDIS_ZSET   | REDIS_ENCODING_SKIPLIST   | 使用跳跃表和字典           | 不满足以上条件                    |                      |                                          |

- TYPE [KEY] 返回对象类型
- OBJECT ENCODING [KEY] 返回编码类型
- raw和embstr编码的区别在于embstr通过一次内存分配分配一块连续地址的内存，来存储redisObject和sdshdr。而raw则是两次分配，分别创建redisObject结构和sdshdr对象。释放亦然。
- 跳跃表和字典结合实现zset，跳跃表按分值从小到大保存所有集合元素，主要用于范围操作。dict字典为有序集合创建一个从元素到分值的映射，用于O(1)获取分值。两者通过指针共享元素的成员和分值。

### 内存回收

- 采用基于引用计数的内存回收机制
```c
typedef struct redisObject{
  // 引用计数
  int refcount;
  // ...
} robj;
```
- OBJECT REFCOUNT [KEY] 查看引用计数
-  根据引用计数实现对象共享。
- 主要为字符串的共享。服务会初始化从0~REDIS_SHARED_INTEGERS之间的数字字符串

### LRU
```c
typedef struct redisObject{
  // 最后一次被访问的时间
  unsigned lru:22;
  // ...
} robj;
```
- OBJECT IDLETIME [KEY] 查看空转时长 = NOW-LRU
- 服务器打开maxmemory时，回收算法为volatile-lru或allkeys-lru时，空转时长较长的优先被释放。
- maxmemory <bytes> 配置最大内存，超过则采用maxmemory-policy配置的策略删除键值，删除失败则对发送的请求返回失败。
- maxmemory-policy 内存饱和策略：
	- volatile-lru 根据LRU算法从有过期时间的键值集合中移除
	- allkeys-lru  根据LRU算法从所有键值集合中移除
	- volatile-random 随机从有过期时间的键值集合中移除
	- allkeys-random 随机从所有键值集合中移除c
	- volatile-ttl 移除即将最快过期的键值
	- noeviction 无移除策略
