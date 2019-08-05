# redis基础原理

## 一、redis简介
redis全称Remote Dictionary Server。Redis本质上是一个Key-Value类型的内存数据库，整个数据库统统加载在内存当中进行操作，定期通过异步操作把数据库数据flush到硬盘上进行保存。Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。 

**Redis 优势**
* 性能极高 – 因为是纯内存操作，Redis能读的速度是110000次/s,写的速度是81000次/s 。
* 丰富的数据类型 – Redis支持 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
* 原子 – Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。
* 丰富的特性 – Redis 还支持 publish/subscribe, 队列，key过期等等特性。
* Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。
* Redis支持数据的备份，即master-slave模式的数据备份。

**Redis的缺点**
* 数据库容量受到物理内存的限制，不能用作海量数据的高性能读写，因此Redis适合的场景主要局限在较小数据量的高性能操作和运算上。

## 二、Redis对象类型及底层结构
### Redis对象
Redis中的一个对象的结构体表示如下
```
typedef struct redisObject {  
  
    // 类型  
    unsigned type:4;          

    // 编码方式  
    unsigned encoding: 4;  
  
    // 引用计数  
    int refcount;  
  
    // 指向对象的值  
    void *ptr;  
  
} robj;
```
type表示了该对象的对象类型，即上面五个中的一个。但为了提高存储效率与程序执行效率，每种对象的底层数据结构实现都可能不止一种。encoding就表示了对象底层所使用的编码。
redis常见的对象类型包括STRING、LIST、SET、HASH、ZSET，具体描述如下表

| 数据类型 | 可以存储的值 | 操作 |
| -------- | -------- | -------- |
|STRING	|字符串、整数或者浮点数|对整个字符串或者字符串的其中一部分执行操作；对整数和浮点数执行自增或者自减操作|
|LIST	|列表	|从两端压入或者弹出元素；读取单个或者多个元素进行修剪；只保留一个范围内的元素|
|SET	|无序集合|添加、获取、移除单个元素；检查一个元素是否存在于集合中；计算交集、并集、差集从集合里面随机获取元素|
|HASH	|包含键值对的无序散列表|添加、获取、移除单个键值；对获取所有键值对检查某个键是否存在|
|ZSET	|有序集合	|添加、获取、删除元素；根据分值范围或者成员来获取元素；计算一个键的排名|

### Redis对象底层数据结构

|编码常量	|编码所对应的底层数据结构|
| -------- | -------- |
|REDIS_ENCODING_INT	|long类型的整数|
|REDIS_ENCODING_EMBSTR|	embstr编码的简单动态字符串|
|REDIS_ENCODING_RAW|简单动态字符串|
|REDIS_ENCODING_HT	|字典|
|REDIS_ENCODING_LINKEDLIST|	双端链表|
|REDIS_ENCODING_ZIPLIST	|压缩列表|
|REDIS_ENCODING_INTSET|	整数集合|
|REDIS_ENCODING_SKIPLIST|	跳跃表和字典|

#### （1）字符串对象
字符串对象的编码可以是int、raw或者embstr如果一个字符串的内容可以转换为long，那么该字符串就会被转换成为long类型，对象的ptr就会指向该long，并且对象类型也用int类型表示。 普通的字符串有embstr和raw两种。embstr应该是Redis 3.0新增的数据结构,在2.8中是没有的。如果字符串对象的长度小于39字节，就用embstr对象。否则用传统的raw对象。
```
#define REDIS_ENCODING_EMBSTR_SIZE_LIMIT 44  
robj *createStringObject(char *ptr, size_t len) {  
    if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT)  
        return createEmbeddedStringObject(ptr,len);  
    else  
        return createRawStringObject(ptr,len);  
}
```
embstr的好处有如下几点：
* embstr的创建只需分配一次内存，而raw为两次（一次为 sds 分配对象，另一次为objet分配对象，embstr省去了第一次）。相对地，释放内存的次数也由两次变为一次。
* embstr的objet和sds放在一起，更好地利用缓存带来的优势。
 
raw和embstr的区别可以用下图所示：
![](https://i.imgur.com/dSaTHBq.png)
![](https://i.imgur.com/6594PK2.png)

#### （2）列表对象 
列表对象的编码可以是ziplist或者linkedlist
1、**ziplist**是一种压缩链表，它的好处是更能节省内存空间，因为它所存储的内容都是在连续的内存区域当中的。当列表对象元素不大，每个元素也不大的时候，就采用ziplist存储但当数据量过大时就ziplist就不是那么好用了。因为为了保证他存储内容在内存中的连续性，插入的复杂度是O(N)，即每次插入都会重新进行realloc。如下图所示，对象结构中ptr所指向的就是一个ziplist整个ziplist只需要malloc一次，它们在内存中是一块连续的区域。
![](https://i.imgur.com/KGHAOpb.png)

2、**linkedlist**是一种双向链表。它的结构比较简单，节点中存放pre和next两个指针，还有节点相关的信息。当每增加一个node的时候，就需要重新malloc一块内存。
![](https://i.imgur.com/P8c4Mb0.png)

当同时满足下面两个条件时，使用ziplist（压缩列表）编码：
* 列表保存元素个数小于512个
* 每个元素长度小于64字节

不能满足这两个条件的时候使用 linkedlist 编码。

#### （3）哈希对象 
哈希对象的底层实现可以是ziplist或者hashtable。 ziplist中的哈希对象是按照key1,value1,key2,value2这样的顺序存放来存储的。当对象数目不多且内容不大时，这种方式效率是很高的。
hashtable的是由dict这个结构来实现的, dict是一个字典，其中的指针dicht ht[2]指向了两个哈希表
```
typedef struct dict {  
    dictType *type;  
    void *privdata;  
    dictht ht[2];  
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */  
    int iterators; /* number of iterators currently running */  
} dict;  
typedef struct dictht {  
    dictEntry **table;  
    unsigned long size;  
    unsigned long sizemask;  
    unsigned long used;  
} dictht;
```
dicht[0] 是用于真正存放数据，dicht[1]一般在哈希表元素过多进行rehash的时候用于中转数据。 dictht中的table用语真正存放元素了，每个key/value对用一个dictEntry表示，放在dictEntry数组中。

![](https://i.imgur.com/0k1VbA6.png)
和上面列表对象使用 ziplist 编码一样，当同时满足下面两个条件时，使用ziplist（压缩列表）编码：
* 列表保存元素个数小于512个
* 每个元素长度小于64字节

不能满足这两个条件的时候使用hashtable编码

#### （4）集合对象 
集合对象的编码可以是intset或者hashtable intset是一个整数集合，里面存的为某种同一类型的整数，支持如下三种长度的整数：
```
#define INTSET_ENC_INT16 (sizeof(int16_t))  
#define INTSET_ENC_INT32 (sizeof(int32_t))  
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
intset是一个有序集合，查找元素的复杂度为O(logN)，但插入时不一定为O(logN)，因为有可能涉及到升级操作。比如当集合里全是int16_t型的整数，这时要插入一个int32_t，那么为了维持集合中数据类型的一致，那么所有的数据都会被转换成int32_t类型，涉及到内存的重新分配，这时插入的复杂度就为O(N)了。 intset不支持降级操作。

当集合同时满足以下两个条件时，使用 intset 编码：
* 集合对象中所有元素都是整数
* 集合对象所有元素数量不超过512

不能满足这两个条件的就使用hashtable编码。第二个条件可以通过配置文件的set-max-intset-entries进行配置。

#### （5）有序集合对象 
有序集合的编码可能两种，一种是ziplist，另一种是skiplist与dict的结合。 ziplist作为集合和作为哈希对象是一样的，member和score顺序存放。按照score从小到大顺序排列。 skiplist是一种跳跃表，它实现了有序集合中的快速查找，在大多数情况下它的速度都可以和平衡树差不多。但它的实现比较简单，可以作为平衡树的替代品。它的结构比较特殊。下面分别是跳跃表skiplist和它内部的节点skiplistNode的结构体：
```
/* 
 * 跳跃表 
 */  
typedef struct zskiplist {  
    // 头节点，尾节点  
    struct zskiplistNode *header, *tail;  
    // 节点数量  
    unsigned long length;  
    // 目前表内节点的最大层数  
    int level;  
} zskiplist;  
/* ZSETs use a specialized version of Skiplists */  
/* 
 * 跳跃表节点 
 */  
typedef struct zskiplistNode {  
    // member 对象  
    robj *obj;  
    // 分值  
    double score;  
    // 后退指针  
    struct zskiplistNode *backward;  
    // 层  
    struct zskiplistLevel {  
        // 前进指针  
        struct zskiplistNode *forward;  
        // 这个层跨越的节点数量  
        unsigned int span;  
    } level[];  
} zskiplistNode;
```
head和tail分别指向头节点和尾节点，然后每个skiplistNode里面的结构又是分层的(即level数组) 用图表示，大概是下面这个样子：
![](https://i.imgur.com/WahqNqE.png)

当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：
* 保存的元素数量小于128
* 保存的所有元素长度都小于64字节

不能满足上面两个条件的使用skiplist编码。以上两个条件也可以通过Redis配置文件zset-max-ziplist-entries选项和 zset-max-ziplist-value进行修改。

## 三、Redis 的持久化方式
### 1、RDB 快照（snapshot）
将存在于某一时刻的所有数据都写入到硬盘中。
#### （1）快照的原理
在默认情况下，Redis 将数据库快照保存在名字为 dump.rdb 的二进制文件中。你可以对 Redis 进行设置， 让它在“N 秒内数据集至少有 M 个改动”这一条件被满足时， 自动保存一次数据集。你也可以通过调用 SAVE 或者 BGSAVE，手动让 Redis 进行数据集保存操作。这种持久化方式被称为快照。
当 Redis 需要保存 dump.rdb 文件时， 服务器执行以下操作:
Redis 创建一个子进程。
子进程将数据集写入到一个临时快照文件中。
当子进程完成对新快照文件的写入时，Redis 用新快照文件替换原来的快照文件，并删除旧的快照文件。
这种工作方式使得 Redis 可以从写时复制（copy-on-write）机制中获益。
#### （2）快照的优点
它保存了某个时间点的数据集，非常适用于数据集的备份。
很方便传送到另一个远端数据中心或者亚马逊的 S3（可能加密），非常适用于灾难恢复。
快照在保存 RDB 文件时父进程唯一需要做的就是 fork 出一个子进程，接下来的工作全部由子进程来做，父进程不需要再做其他 IO 操作，所以快照持久化方式可以最大化 redis 的性能。
与 AOF 相比，在恢复大的数据集的时候，DB 方式会更快一些。
#### （3）快照的缺点
如果你希望在 redis 意外停止工作（例如电源中断）的情况下丢失的数据最少的话，那么快照不适合你。
快照需要经常 fork 子进程来保存数据集到硬盘上。当数据集比较大的时候，fork 的过程是非常耗时的，可能会导致 Redis 在一些毫秒级内不能响应客户端的请求。

### 2、AOF
AOF 持久化方式记录每次对服务器执行的写操作。当服务器重启的时候会重新执行这些命令来恢复原始的数据。
 
#### （1）AOF的原理
Redis 创建一个子进程。子进程开始将新 AOF 文件的内容写入到临时文件。对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾，这样样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。
当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。
搞定！现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。
 
#### （2）AOF的优点
使用默认的每秒 fsync 策略，Redis 的性能依然很好(fsync 是由后台线程进行处理的,主线程会尽力处理客户端请求)，一旦出现故障，使用 AOF ，你最多丢失 1 秒的数据。
AOF 文件是一个只进行追加的日志文件，所以不需要写入 seek，即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令，你也也可使用 redis-check-aof 工具修复这些问题。
Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写：重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。整个重写操作是绝对安全的。
AOF 文件有序地保存了对数据库执行的所有写入操作，这些写入操作以 Redis 协议的格式保存。因此 AOF 文件的内容非常容易被人读懂，对文件进行分析（parse）也很轻松。
 
#### （3）AOF的缺点
对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。根据所使用的 fsync 策略，AOF 的速度可能会慢于快照。在一般情况下，每秒 fsync 的性能依然非常高，而关闭 fsync 可以让 AOF 的速度和快照一样快，即使在高负荷之下也是如此。不过在处理巨大的写入载入时，快照可以提供更有保证的最大延迟时间（latency）。

## 四、Redis过期键的删除
### 1、过期策略
* 定时删除:在设置键的过期时间的同时，创建一个定时器timer. 让定时器在键的过期时间来临时，立即执行对键的删除操作。
* 惰性删除:放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键;如果没有过期，就返回该键。
* 定期删除:每隔一段时间程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

### 2、回收策略
* noeviction - 当内存使用达到阈值的时候，所有引起申请内存的命令会报错。
* allkeys-lru - 在主键空间中，优先移除最近未使用的 key。
* allkeys-random - 在主键空间中，随机移除某个 key。
* volatile-lru - 在设置了过期时间的键空间中，优先移除最近未使用的 key。
* volatile-random - 在设置了过期时间的键空间中，随机移除某个 key。
* volatile-ttl - 在设置了过期时间的键空间中，具有更早过期时间的 key 优先移除。

注意这里的6种机制，volatile和allkeys规定了是对已设置过期时间的数据集淘汰数据还是从全部数据集淘汰数据，后面的lru、ttl以及random是三种不同的淘汰策略，再加上一种no-enviction永不回收的策略。
**使用策略规则：**
* 如果数据呈现幂律分布，也就是一部分数据访问频率高，一部分数据访问频率低，则使用allkeys-lru
* 如果数据呈现平等分布，也就是所有的数据访问频率都相同，则使用allkeys-random

## 五、redis高性能
单线程的Redis如何保证性能
**Redis 快速的原因：**
* 绝大部分请求是纯粹的内存操作（非常快速）
* 采用单线程,避免了不必要的上下文切换和竞争条件
* 非阻塞 IO内部实现采用 epoll，采用了 epoll+自己实现的简单的事件框架。epoll 中的读、写、关闭、连接都转化成了事件，然后利用 epoll 的多路复用特性，绝不在 io 上浪费一点时间。

**常见性能保障措施：**
* Master最好不要写内存快照，如果Master写内存快照，save命令调度rdbSave函数，会阻塞主线程的工作，当快照比较大时对性能影响是非常大的，会间断性暂停服务
* 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次
* 为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网
* 尽量避免在压力很大的主库上增加从
* 主从复制不要用图状结构，用单向链表结构更为稳定，即：Master <- Slave1 <- Slave2 <- Slave3…这样的结构方便解决单点故障问题，实现Slave对Master的替换。如果Master挂了，可以立刻启用Slave1做Master，其他不变。

## 六、Redis的key寻址
### 背景
* （1）redis中的每一个数据库，都由一个redisDb的结构存储。其中：
redisDb.id存储着 redis 数据库以整数表示的号码；
redisDb.dict 存储着该库所有的键值对数据；
redisDb.expires 保存着每一个键的过期时间。
* （2）当 redis 服务器初始化时，会预先分配16个数据库（该数量可以通过配置文件配置），所有数据库保存到结构 redisServer 的一个成员redisServer.db数组中。当我们选择数据库select number时，程序直接通过 redisServer.db[number]来切换数据库。有时候当程序需要知道自己是在哪个数据库时，直接读取redisDb.id即可。
* （3）redis的字典使用哈希表作为其底层实现。dict类型使用的两个指向哈希表的指针，其中0号哈希表（ht[0]）主要用于存储数据库的所有键值，而1号哈希表主要用于程序对0号哈希表进行 rehash时使用，rehash一般是在添加新值时会触发，这里不做过多的赘述。所以redis中查找一个key，其实就是对进行该dict结构中的ht[0]进行查找操作。
* （4）既然是哈希，那么我们知道就会有哈希碰撞，那么当多个键哈希之后为同一个值怎么办呢？redis 采取链表的方式来存储多个哈希碰撞的键。也就是说，当根据 key 的哈希值找到该列表后，如果列表的长度大于 1，那么我们需要遍历该链表来找到我们所查找的 key。当然，一般情况下链表长度都为是1，所以时间复杂度可看作o(1)。
### key寻址的步骤
* （1）当拿到一个key后，redis先判断当前库的0号哈希表是否为空，即：if (dict->ht[0].size == 0)。如果为true直接返回NULL。
* （2）判断该0号哈希表是否需要rehash，因为如果在进行 rehash，那么两个表中者有可能存储该 key。如果正在进行 rehash，将调用一次_dictRehashStep方法，_dictRehashStep 用于对数据库字典、以及哈希键的字典进行被动 rehash，这里不作赘述。
* （3）计算哈希表，根据当前字典与 key 进行哈希值的计算。
* （4）根据哈希值与当前字典计算哈希表的索引值。
* （5）根据索引值在哈希表中取出链表，遍历该链表找到key的位置。一般情况，该链表长度为 1。
* （6）当ht[0]查找完了之后，再进行了次rehash判断，如果未在rehashing，则直接结束，否则对 ht[1]重复345步骤。

## 七、Redis的主从复制
### 1、旧版本全量复制功能的实现
![](https://i.imgur.com/sWztnq2.png)
全量复制使用 Snyc 命令来实现，其流程是：
* 从服务器向主服务器发送 Sync 命令。
* 主服务器在收到 Sync 命令之后，调用 Bgsave 命令生成最新的 RDB 文件，将这个文件同步给从服务器，这样从服务器载入这个 RDB 文件之后，状态就会和主服务器执行 Bgsave 命令时候的一致。
* 主服务器将保存在命令缓冲区中的写命令同步给从服务器，从服务器执行这些命令，这样从服务器的状态就跟主服务器当前状态一致了。

旧版本全量复制功能，其最大的问题是从服务器断线重连时，即便在从服务器上已经有一部分数据了，也需要进行全量复制，这样做的效率很低，于是新版本的 Redis 在这部分做了改进。

### 2、新版本全量复制功能的实现
新版本 Redis 使用 Psync 命令来代替 Sync 命令，该命令既可以实现完整全同步也可以实现部分同步。
新的流程：
* 从服务器连接主服务器，发送 SYNC 命令；
* 主服务器接收到 SYNC 命名后，开始执行 BGSAVE 命令生成 RDB 文件并使用缓冲区记录此后执行的所有写命令；
* 主服务器 BGSAVE 执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令；
* 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照；
* 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令；
* 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；


### 3、复制偏移量

执行复制的双方，主从服务器，分别会维护一个复制偏移量：
* 主服务器每次向从服务器同步了 N 字节数据之后，将修改自己的复制偏移量 +N。
* 从服务器每次从主服务器同步了 N 字节数据之后，将修改自己的复制偏移量 +N。


### 4、复制积压缓冲区
![](https://i.imgur.com/7fiDDOQ.png)
主服务器内部维护了一个固定长度的先进先出队列做为复制积压缓冲区，其默认大小为 1MB。
在主服务器进行命令传播时，不仅会将写命令同步到从服务器，还会将写命令写入复制积压缓冲区。


### 5、服务器运行ID
每个 Redis 服务器，都有其运行 ID，运行 ID 由服务器在启动时自动生成，主服务器会将自己的运行 ID 发送给从服务器，而从服务器会将主服务器的运行 ID 保存起来。

从服务器 Redis 断线重连之后进行同步时，就是根据运行 ID 来判断同步的进度：
* 如果从服务器上面保存的主服务器运行 ID 与当前主服务器运行 ID 一致，则认为这一次断线重连连接的是之前复制的主服务器，主服务器可以继续尝试部分同步操作。
* 否则，如果前后两次主服务器运行 ID 不相同，则认为是完成全同步流程。


### 6、Psync命令流程
有了前面的准备，下面开始分析 Psync 命令的流程：

* 如果从服务器之前没有复制过任何主服务器，或者之前执行过 slaveof no one 命令，那么从服务器就会向主服务器发送 psync ? -1 命令，请求主服务器进行数据的全量同步。
* 否则，如果前面从服务器已经同步过部分数据，那么从服务器向主服务器发送 psync <runid> <offset> 命令，其中 runid 是上一次主服务器的运行 id，offset 是当前从服务器的复制偏移量。
![](https://i.imgur.com/TDINvcV.png)


前面两种情况主服务器收到 Psync 命令之后，会出现以下三种可能：
* 主服务器返回 +fullresync <runid> <offset> 回复，表示主服务器要求与从服务器进行完整的数据全量同步操作。
* 其中，runid 是当前主服务器运行 id，而 offset 是当前主服务器的复制偏移量。
* 如果主服务器应答 +continue，那么表示主服务器与从服务器进行部分数据同步操作，将从服务器缺失的数据同步过来即可。
* 如果主服务器应答 -err，那么表示主服务器版本低于 2.8，识别不了 Psync 命令，此时从服务器将向主服务器发送 Sync 命令，执行完整的全量数据同步。


## 八、Redis集群模式
### 1、Redis Cluster
Redis Cluster是Redis的分布式解决方案，在 Redis 3.0版本正式推出的。Redis Cluster去中心化，每个节点保存数据和整个集群状态，每个节点都和其他所有节点连接。
#### 1.1、Redis Cluster节点分配
Redis Cluster特点：
所有的redis节点彼此互联(PING-PONG 机制)，内部使用二进制协议优化传输速度和带宽。节点的 fail 是通过集群中超过半数的节点检测失效时才生效。客户端与 redis 节点直连,不需要中间 proxy 层。客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可。
redis-cluster 把所有的物理节点映射到0-16383哈希槽 (hash slot)上（不一定是平均分配）,cluster 负责维护 node、slot、value。
Redis集群预分好 16384 个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384 的值，决定将一个 key 放到哪个桶中。
#### 1.2、Redis Cluster主从模式
Redis Cluster 为了保证数据的高可用性，加入了主从模式。
一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份。当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉。所以，在集群建立的时候，一定要为每个主节点都添加了从节点。

#### 1.3、Redis Cluster原理
##### 1.3.1、CLUSTER MEET命令的实现

通过向节点A发送CLUSTER MEET命令，客户端可以让接收命令的节点A将另一个节点B添加到节点A当前所在的集群里面：
CLUSTER MEET <ip> <port>
收到命令的节点A将与节点B进行握手（handshake），以此来确认彼此的存在，并为将来的进一步通信打好基础：

* 1）节点A会为节点B创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。
* 2）之后，节点A将根据CLUSTER MEET命令给定的IP地址和端口号，向节点B发送一条MEET消息。
* 3）如果一切顺利，节点B将接收到节点A发送的MEET消息，节点 B 会为节点A创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。
* 4）之后，节点 B 将向节点 A 返回一条 PONG 消息。
* 5）如果一切顺利，节点A将收到节点B返回的PONG消息，通过这条PONG消息节点A可以知道节点B已经成功地接收到了自己发送的MEET消息。
* 6）之后，节点A将向节点B返回一条PING消息。
* 7）如果一切顺利，节点B将接收到节点A返回的PING消息，通过这条PING消息节点B可以知道节点A已经成功地接收到了自己返回的PONG消息，握手完成。

之后，节点A会将节点的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终，经过一段时间后，节点B会被集群中的所有节点认识。

##### 1.3.2、槽指派
Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。

当数据库中的16384个槽都有节点在处理时，集群处于上线状态（ok）；相反地，如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态（fail）。

通过向节点发送CLUSTER ADDSLOTS命令，可以将一个或多个槽指派（assign）给节点负责：
```
CLUSTER ADDSLOTS <slot> [slot . . .]

127.0.0.1:7000> CLUSTER ADDSLOTS 0 1 2 3 4 . . . 5000

OK

127.0.0.1:7000> CLUSTER INFO

cluster_state:ok
```

clusterNode的slots属性和numslot属性记录了节点负责处理哪些槽：
```
struct clusterNode {

// ...

unsigned char slots[16384/8];

int numslots;

// ...

};
```

slots属性是一个二进制位数组（bit array），这个数组的长度为2048个字节，共包含16384个二进制位。如果slots数组在索引i上的二进制位的值为1，那么表示节点负责处理槽i，为0表示不负责。

一个节点除了会将自己负责处理的槽记录在clusterNode结构的slots属性和numslots属性之外，它还会将自己的slots数组通过消息发送给集群中的其他节点，以此来告诉其他节点自己目前负责处理哪些槽。

clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息：
```
typedef struct clusterState {

// ...

clusterNode *slots[16384]; // 每个数组项指向一个clusterNode

// ...

} clusterState;
```

##### 1.3.3、在集群中执行命令

在对数据库中的16384个槽都进行了指派之后，集群就会进入上线状态，这时客户端就可以向集群中的节点发送数据命令了。

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

如果指派给了当前节点，节点直接执行这个命令。否则，节点会向客户端返回一个MOVED错误，指引客户端转向（redirect）至正确的节点，并再次发送之前想要执行的命令。

计算键属于那个槽：

```
def slot_number(key):
return CRC16(key) & 16383
```

// CRC-16校验和

判断槽i是否由当前节点负责处理：
```
clusterState.slots[i] == clusterState.myself
```

一个集群客户端通常会与集群中的多个节点创建套接字连接，而所谓的节点转向实际上就是换一个套接字来发送命令。

节点和单机服务器在数据库方面的一个区别是，节点只能使用0号数据库。

##### 1.3.4、重新分片

Redis集群的重新分片操作可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点移动到目标节点。（这里的重新分片不是rehash，请注意与客户端一致性hash分片区分开来）

重新分片操作可以在线进行，在重新分片过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。

重新分片操作由Redis的集群管理软件redis-trib负责执行，redis提供了进行重新分片所需的所有命令，而redis-trib则通过想源节点和目标节点发送命令来进行重新分片操作。

redis-trib对集群的单个槽slot进行重新分片的步骤如下：

* 1）redis-trib对目标节点发送CLUSTER SETSLOT <slot> IMPORTING <source_id>命令，让目标节点准备好从源节点导入（import）属于槽slot的键值对。
* 2）redis-trib对源节点发送CLUSTER SETSLOT <slot> MIGRATING <target_id>命令，让源节点准备好将属于槽slot的键值对迁移（migrate）至目标节点。
* 3）redis-trib向源节点发送CLUSTER GETKEYSINSLOT <slot> <count>命令，获得最多count个属于槽slot的键值对的键名。
* 4）对于步骤3获得的每个键名，redis-trib都向源节点发送一个MIGRATE <target_ip> <target_port> <key_name> 0 <timeout>命令，将被选中的键原子地从源节点迁移至目标节点。
* 5）重复执行步骤3和步骤4，直到源节点保存的所有属于槽slot的键值对都被迁移至目标节点为止。
* 6）redis-trib向集群中的任意一个节点发送CLUSTER SETSLOT <slot> NODE <target_id>命令，将槽slot指派给目标节点，这一指派信息会通过消息发送至整个集群，最终集群中的所有节点都会知道槽slot已经被指派给了目标节点。

**ASK错误：**

在进行重新分片期间，源节点向目标节点迁移一个槽的过程中，可能会出现这样一种情况：属于被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。

当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令。没找到的话，那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

##### 1.3.5、复制与故障转移

Redis集群中的节点分为主节点（master）和从节点（slave），其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。
设置从节点CLUSTER REPLICATE<node_id>

**故障检测：**
集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此来检测对方是否在线，如果接收PING消息的节点没有在规定的时间内返回PONG消息，那么发送PING消息的节点就会将接收PING消息的节点标记为疑似下线（probable fail, PFAIL）。

如果在一个集群里面，半数以上负责处理槽的节点都将某个主节点x报告为疑似下线，那么这个主节点x将被标记为已下线（FAIL），将x标记为FAIL的节点会向集群广播一条关于x的FAIL消息，所有收到这条FAIL消息的节点都会立即将x标记为FAIL。

**故障转移：**

当一个从节点发现自己正在复制的主节点进入FAIL状态时，从节点将开始对下线主节点进行故障转移，以下是故障转移的执行步骤：

* 1）复制下线主节点的所有从节点里面，会有一个从节点被选中
* 2）被选中的从节点会执行SLAVEOF no one命令，称为新的主节点
* 3）新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己
* 4）新的主节点向集群广播一条PONG消息，这条PONG消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽
* 5）新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成

**选举新的主节点：**

* 1）集群的配置纪元是一个自增计数器，它的初始值为0
* 2）当集群里的某个节点开始一次故障转移操作时，集群配置纪元的值会被增一
* 3）对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票 
* 4）档从节点发现自己正在复制的主节点进入已下线状态时，从节点会想集群广播一条CLUSTER_TYPE_FAILOVER_AUTH_REQUEST消息，要求所有接收到这条消息、并且具有投票权的主节点向这个从节点投票。
* 5）如果一个主节点具有投票权（它正在负责处理槽），并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，表示这个主节点支持从节点成为新的主节点。
* 6）每个参与选举的从节点都会接收CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，并根据自己收到了多少条这种消息来同济自己获得了多少主节点的支持。
* 7）如果集群里有N个具有投票权的主节点，那么当一个从节点收集到大于等于N/2+1张支持票时，这个从节点就会当选为新的主节点。
* 8）因为在每一个配置纪元里面，每个具有投票权的主节点只能投一次票，所以如果有N个主节点进行投票，那么具有大于等于N/2+1张支持票的从节点只会有一个，这确保了新的主节点只会有一个。
* 9）如果在一个配置纪元里面没有从节点能收集到足够多的支持票，那么集群进入一个新的配置纪元，并再次进行选举，知道选出新的主节点为止

这个选举新主节点的方法和选举领头Sentinel的方法非常相似，因为两者都是基于Raft算法的领头选举方法来实现的。

### 2、Redis Sentinel
#### 2.1、Redis Sentinel概述
Redis Sentinel用于管理多个Redis服务器，主要有三个功能：
* （1）监控（Monitoring） - Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
* （2）提醒（Notification） - 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
* （3）自动故障迁移（Automatic failover） - 当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

**Redi哨兵机制的大概工作原理**：
Redis 使用一组哨兵（Sentinel）节点来监控主从 Redis 服务的可用性。
一旦发现 Redis 主节点失效，将选举出一个哨兵节点作为领导者（Leader）。
哨兵领导者再从剩余的从 Redis 节点中选出一个 Redis 节点作为新的主 Redis 节点对外服务。

**Redis节点分类**：
* 哨兵节点（Sentinel）：负责监控节点的运行情况。
* 数据节点：即正常服务客户端请求的 Redis 节点，有主从之分。

#### 2.2、三个监控任务

哨兵节点通过三个定时监控任务监控 Redis 数据节点的服务可用性。

①**info命令**
![](https://i.imgur.com/qy2u0IN.png)

每隔10秒，每个哨兵节点都会向主、从 Redis 数据节点发送info命令，获取新的拓扑结构信息。

Redis 拓扑结构信息包括了：
* 本节点角色：主或从。
* 主从节点的地址、端口信息。

这样，哨兵节点就能从 info 命令中自动获取到从节点信息，因此那些后续才加入的从节点信息不需要显式配置就能自动感知。


②**向 sentinel:hello频道同步信息**
![](https://i.imgur.com/oZKdpEa.png)

每隔2秒，每个哨兵节点将会向 Redis 数据节点的 __sentinel__:hello 频道同步自身得到的主节点信息以及当前哨兵节点的信息。

由于其他哨兵节点也订阅了这个频道，因此实际上这个操作可以交换哨兵节点之间关于主节点以及哨兵节点的信息。

这一操作实际上完成了两件事情：
* 发现新的哨兵节点：如果有新的哨兵节点加入，此时保存下来这个新哨兵节点的信息，后续与该哨兵节点建立连接。
* 交换主节点的状态信息，作为后续客观判断主节点下线的依据。


③**向数据节点做心跳探测**
![](https://i.imgur.com/0pVpt58.png)

每隔1秒，每个哨兵节点向主、从数据节点以及其他Sentinel节点发送 Ping 命令做心跳探测，这个心跳探测是后续主观判断数据节点下线的依据。


#### 2.3、主观下线和客观下线

* ①主观下线
![](https://i.imgur.com/afGLUOo.png)

上面三个监控任务中的第三个探测心跳任务，如果在配置的 down-after-milliseconds 之后没有收到有效回复，那么就认为该数据节点“主观下线（sdown）”。
为什么称为“主观下线”？因为在一个分布式系统中，有多个机器在一起联动工作，网络可能出现各种状况，仅凭一个节点的判断还不足以认为一个数据节点下线了，这就需要后面的“客观下线”。



* ②客观下线
![](https://i.imgur.com/u2Fzh6m.png)

当一个哨兵节点认为主节点主观下线时，该哨兵节点需要通过”sentinel is-master-down-by addr”命令向其他哨兵节点咨询该主节点是否下线了，如果有超过半数的哨兵节点都回答了下线，此时认为主节点“客观下线”。



#### 2.4、选举哨兵领导者
![](https://i.imgur.com/Nl5cgr7.png)

当主节点客观下线时，需要选举出一个哨兵节点做为哨兵领导者，以完成后续选出新的主节点的工作。

**选举的大体思路**：
* 每个哨兵节点通过向其他哨兵节点发送”sentinel is-master-down-by addr”命令来申请成为哨兵领导者。
* 而每个哨兵节点在收到一个”sentinel is-master-down-by addr”命令时，只允许给第一个节点投票，其他节点的该命令都会被拒绝。
* 如果一个哨兵节点收到了半数以上的同意票，则成为哨兵领导者。
* 如果前面三步在一定时间内都没有选出一个哨兵领导者，将重新开始下一次选举。

可以看到，这个选举领导者的流程很像 Raft 中选举 Leader 的流程。

#### 2.5、选出新的主节点
![](https://i.imgur.com/nXJuud9.png)

在剩下的 Redis 从节点中，按照以下顺序来选择新的主节点：
* 过滤掉“不健康”的数据节点：比如主观下线、断线的从节点、五秒内没有回复过哨兵节点 Ping 命令的节点、与主节点失联的从节点。
* 选择 Slave-Priority（从节点优先级）最高的从节点，如果存在则返回，不存在则继续后面的流程。
* 选择复制偏移量最大的从节点，这意味着这个从节点上面的数据最完整，如果存在则返回，不存在则继续后面的流程。
* 到了这里，所有剩余从节点的状态都是一样的，选择 runid 最小的从节点。



#### 2.6、提升新的主节点
![](https://i.imgur.com/BW5AGTb.png)

选择了新的主节点之后，还需要最后的流程让该节点成为新的主节点：
* 哨兵领导者向上一步选出的从节点发出“slaveof no one”命令，让该节点成为主节点。
* 哨兵领导者向剩余的从节点发送命令，让它们成为新主节点的从节点。
* 哨兵节点集合会将原来的主节点更新为从节点，当其恢复之后命令它去复制新的主节点的数据。

 
## 九、Redis实现分布式锁
### 1、redis分布式锁的基本介绍
实现分布式锁的三种选择：
* 基于数据库实现分布式锁
* 基于zookeeper实现分布式锁
* 基于Redis缓存实现分布式锁 

以上三种方式都可以实现分布式锁，其中，从健壮性考虑， 用zookeeper会比用Redis实现更好，但从性能角度考虑，基于 Redis实现性能会更好，如何选择，还是取决于业务需求。

基于Redis实现分布式锁的三种方案：
* 用Redis实现分布式锁的正确姿势（实现一）
* 用Redisson实现分布式可重入锁RedissonLock（实现二）
* 用Redisson实现分布式锁红锁RedissonRedLock（实现三）

分布式锁需满足四个条件
* 互斥性。在任意时刻，只有一个客户端能持有锁。
* 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。
* 解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了，即不能误解锁。
* 具有容错性。只要大多数Redis节点正常运行，客户端就能够获取和释放锁。

实现Redis实现分布式锁的一般步骤：
* 获取锁的时候，使用setnx加锁，并使用expire命令为锁添加一个超时时间，超过该时间则自动释放锁，锁的value值为一个随机生成的UUID，通过此在释放锁的时候进行判断。
* 获取锁的时候还设置一个获取的超时时间，若超过这个时间则放弃获取锁。
* 释放锁的时候，通过UUID判断是不是该锁，若是该锁，则执行delete进行锁释放。

### 2、用Redis实现分布式锁的正确姿势（实现一）
**主要思路**：通过set key value px milliseconds nx 命令实现加锁， 通过Lua脚本实现解锁。
核心实现命令如下：
```
//获取锁（unique_value可以是UUID等）
SET resource_name unique_value NX PX  30000

//释放锁（lua脚本中，一定要比较value，防止误解锁）
if redis.call("get",KEYS[1]) == ARGV[1] then
return redis.call("del",KEYS[1])
else
return 0
end
```
这种实现方式主要有以下几个要点： 
* set命令要用set key value px milliseconds nx，替代 setnx + expire需要分两次执行命令的方式，保证了原子性
* value要具有唯一性，可以使用UUID.randomUUID().toString()方法生成，用来标识这把锁是属于哪个请求加的，在解锁的时候就可以有依据
* 释放锁时要验证value值，防止误解锁。
* 通过Lua脚本来避免Check And Set模型的并发问题，因为在释放锁的时候因为涉及到多个Redis操作（利用了eval命令执行Lua脚本的原子性）

完整代码实现如下：
```
public class RedisTool {
private static final String LOCK_SUCCESS = "OK";
private static final String SET_IF_NOT_EXIST = "NX";
private static final String SET_WITH_EXPIRE_TIME = "PX";
private static final Long RELEASE_SUCCESS = 1L;

/**
 * 获取分布式锁(加锁代码)
 * @param jedis Redis客户端
 * @param lockKey 锁
 * @param requestId 请求标识
 * @param expireTime 超期时间
 * @return 是否获取成功
 */
public static boolean getDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {

    String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);

    if (LOCK_SUCCESS.equals(result)) {
        return true;
    }
    return false;
}

/**
 * 释放分布式锁(解锁代码)
 * @param jedis Redis客户端
 * @param lockKey 锁
 * @param requestId 请求标识
 * @return 是否释放成功
 */
public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

    String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else               return 0 end";

    Object result = jedis.eval(script, Collections.singletonList(lockKey), C                                                   ollections.singletonList(requestId));

    if (RELEASE_SUCCESS.equals(result)) {
        return true;
    }
    return false;

  }
}
```
加锁代码分析
* 首先，set()加入了NX参数，可以保证如果已有key存在，则函数不会调用成功，也就是只有一个客户端能持有锁，满足互斥性。
* 其次，由于我们对锁设置了过期时间，即使锁的持有者后续发生崩溃而没有解锁，锁也会因为到了过期时间而自动解锁（即key被删除），不会发生死锁。
* 最后，因为我们将value赋值为requestId，用来标识这把锁是属于哪个请求加的，那么在客户端在解锁的时候就可以进行校验是否是同一个客户端。

**解锁代码分析**
将Lua代码传到jedis.eval()方法里，并使参数KEYS[1]赋值为lockKey，ARGV[1]赋值为requestId。
在执行的时候，首先会获取锁对应的value值，检查是否与requestId相等，如果相等则解锁（删除key）。
这种方式仍存在单点风险，以上实现在 Redis 正常运行情况下是没问题的，但如果存储锁对应key的那个节点挂了的话，就可能存在丢失锁的风险，导致出现多个客户端持有锁的情况，这样就不能实现资源的独享了。
* 客户端A从master获取到锁
* 在master将锁同步到slave之前，master宕掉了（Redis的主从同步通常是异步的）
* 主从切换，slave节点被晋级为master节点
* 客户端B取得了同一个资源被客户端A已经获取到的另外一个锁。导致存在同一时刻存在不止一个线程获取到锁的情况

所以在这种实现之下，不论Redis的部署架构是单机模式、主从模式、哨兵模式还是集群模式，都存在这种风险，因为Redis的主从同步是异步的。 

幸运的是，Redis之父antirez提出了redlock算法 可以解决这个问题。

### 3、Redisson实现分布式可重入锁及源码分析RedissonLock（实现二）
Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。

它不仅提供了一系列的分布式的Java常用对象，还实现了可重入锁（Reentrant Lock）、公平锁（Fair Lock）、联锁（MultiLock）、 红锁（RedLock）、 读写锁（ReadWriteLock）等，还提供了许多分布式服务。

Redisson提供了使用Redis的最简单和最便捷的方法。宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

**Redisson分布式重入锁用法**
Redisson支持单点模式、主从模式、哨兵模式、集群模式，这里以单点模式为例：

```
// 1.构造redisson实现分布式锁必要的Config
Config config = new Config();
config.useSingleServer().setAddress("redis://127.0.0.1:5379").setPassword("123456").setDatabase(0);
// 2.构造RedissonClient
RedissonClient redissonClient = Redisson.create(config);
// 3.获取锁对象实例（无法保证是按线程的顺序获取到）
RLock rLock = redissonClient.getLock(lockKey);
try {
/**
* 4.尝试获取锁
* waitTimeout 尝试获取锁的最大等待时间，超过这个值，则认为获取锁失败
* leaseTime   锁的持有时间,超过这个时间锁会自动失效（值应设置为大于业务处理的时间，确保在锁有效期内业务能处理完）
*/
boolean res = rLock.tryLock((long)waitTimeout, (long)leaseTime, TimeUnit.SECONDS);
if (res) {
//成功获得锁，在这里处理业务
}
} catch (Exception e) {
throw new RuntimeException("aquire lock fail");
}finally{
//无论如何, 最后都要解锁
rLock.unlock();
}
```
**加锁源码分析**

（1）通过getLock方法获取对象
```
org.redisson.Redisson#getLock()
@Override
public RLock getLock(String name) {
/**
*  构造并返回一个 RedissonLock 对象
* commandExecutor: 与 Redis 节点通信并发送指令的真正实现。需要说明一下，CommandExecutor 实现是通过 eval 命令来执行 Lua 脚本
* name: 锁的全局名称
* id: Redisson 客户端唯一标识，实际上就是一个 UUID.randomUUID()
*/
return new RedissonLock(commandExecutor, name, id);
}
```
（2）、通过tryLock方法尝试获取锁 
tryLock方法里的调用关系大致如下：
![](https://i.imgur.com/VZCVILL.png)

```
org.redisson.RedissonLock#tryLock    

@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
//取得最大等待时间
long time = unit.toMillis(waitTime);
//记录下当前时间
long current = System.currentTimeMillis();
//取得当前线程id（判断是否可重入锁的关键）
long threadId = Thread.currentThread().getId();
//1.尝试申请锁，返回还剩余的锁过期时间
Long ttl = tryAcquire(leaseTime, unit, threadId);
//2.如果为空，表示申请锁成功
if (ttl == null) {
return true;
}
//3.申请锁的耗时如果大于等于最大等待时间，则申请锁失败
time -= System.currentTimeMillis() - current;
if (time Completed ,通知 Future 异步执行已完成
*/
acquireFailed(threadId);
return false;
}
    current = System.currentTimeMillis();

    /**
     * 4.订阅锁释放事件，并通过await方法阻塞等待锁释放，有效的解决了无效的锁申请浪费资源的问题：
     * 基于信息量，当锁被其它资源占用时，当前线程通过 Redis 的 channel 订阅锁的释放事件，一旦锁释放会发消息通知待等待的线程进行竞争
     * 当 this.await返回false，说明等待时间已经超出获取锁最大等待时间，取消订阅并返回获取锁失败
     * 当 this.await返回true，进入循环尝试获取锁
     */
    RFuture&lt;RedissonLockEntry&gt; subscribeFuture = subscribe(threadId);
    //await 方法内部是用CountDownLatch来实现阻塞，获取subscribe异步执行的结果（应用了Netty 的 Future）
    if (!await(subscribeFuture, time, TimeUnit.MILLISECONDS)) {
        if (!subscribeFuture.cancel(false)) {
            subscribeFuture.onComplete((res, e) -&gt; {
                if (e == null) {
                    unsubscribe(subscribeFuture, threadId);
                }
            });
        }
        acquireFailed(threadId);
        return false;
    }

    try {
        //计算获取锁的总耗时，如果大于等于最大等待时间，则获取锁失败
        time -= System.currentTimeMillis() - current;
        if (time &lt;= 0) {
            acquireFailed(threadId);
            return false;
        }

        /**
         * 5.收到锁释放的信号后，在最大等待时间之内，循环一次接着一次的尝试获取锁
         * 获取锁成功，则立马返回true，
         * 若在最大等待时间之内还没获取到锁，则认为获取锁失败，返回false结束循环
         */
        while (true) {
            long currentTime = System.currentTimeMillis();
            // 再次尝试申请锁
            ttl = tryAcquire(leaseTime, unit, threadId);
            // 成功获取锁则直接返回true结束循环
            if (ttl == null) {
                return true;
            }

            //超过最大等待时间则返回false结束循环，获取锁失败
            time -= System.currentTimeMillis() - currentTime;
            if (time &lt;= 0) {
                acquireFailed(threadId);
                return false;
            }

            /**
             * 6.阻塞等待锁（通过信号量(共享锁)阻塞,等待解锁消息）：
             */
            currentTime = System.currentTimeMillis();
            if (ttl &gt;= 0 &amp;&amp; ttl &lt; time) {
                //如果剩余时间(ttl)小于wait time ,就在 ttl 时间内，从Entry的信号量获取一个许可(除非被中断或者一直没有可用的许可)。
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                //则就在wait time 时间范围内等待可以通过信号量
                getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }

            //7.更新剩余的等待时间(最大等待时间-已经消耗的阻塞时间)
            time -= System.currentTimeMillis() - currentTime;
            if (time &lt;= 0) {
                acquireFailed(threadId);
                return false;
            }
        }
    } finally {
        //7.无论是否获得锁,都要取消订阅解锁消息
        unsubscribe(subscribeFuture, threadId);
    }
}
```
其中tryAcquire内部通过调用tryLockInnerAsync实现申请锁的逻辑，申请锁并返回锁有效期还剩余的时间。

如果为空说明锁未被其它线程申请则直接获取并返回，如果获取到时间，则进入等待竞争逻辑。

org.redisson.RedissonLock#tryLockInnerAsync
加锁流程图：
![](https://i.imgur.com/IkipfXH.jpg)

**实现源码：**

```
RFuture tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand command) {
internalLockLeaseTime = unit.toMillis(leaseTime);
    /**
     * 通过 EVAL 命令执行 Lua 脚本获取锁，保证了原子性
     */
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              // 1.如果缓存中的key不存在，则执行 hset 命令(hset key UUID+threadId 1),然后通过 pexpire 命令设置锁的过期时间(即锁的租约时间)
              // 返回空值 nil ，表示获取锁成功
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
               // 如果key已经存在，并且value也匹配，表示是当前线程持有的锁，则执行 hincrby 命令，重入次数加1，并且设置失效时间
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
               //如果key已经存在，但是value不匹配，说明锁已经被其他线程持有，通过 pttl 命令获取锁的剩余存活时间并返回，至此获取锁失败
              "return redis.call('pttl', KEYS[1]);",
               //这三个参数分别对应KEYS[1]，ARGV[1]和ARGV[2]
               Collections.&lt;Object&gt;singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```
参数说明： 
* KEYS[1]就是Collections.singletonList(getName())，表示分布式锁的key
* ARGV[1]就是internalLockLeaseTime，即锁的租约时间（持有锁的有效时间），默认30s
* ARGV[2]就是getLockName(threadId)，是获取锁时set的唯一值 value，即UUID+threadId

**解锁源码分析**
unlock内部通过 get(unlockAsync(Thread.currentThread().getId())) 调用 unlockInnerAsync 解锁。
```
org.redisson.RedissonLock#unlock 

@Override
public void unlock() {
try {
get(unlockAsync(Thread.currentThread().getId()));
} catch (RedisException e) {
if (e.getCause() instanceof IllegalMonitorStateException) {
throw (IllegalMonitorStateException) e.getCause();
} else {
throw e;
}
}
}
```
get方法是利用 CountDownLatch 在异步调用结果返回前将当前线程阻塞。

然后通过Netty的FutureListener在异步调用完成后解除阻塞，并返回调用结果。
```
org.redisson.command.CommandAsyncService#get 

@Override
public  V get(RFuture future) {
if (!future.isDone()) {   //任务还没完成
// 设置一个单线程的同步控制器
CountDownLatch l = new CountDownLatch(1);
future.onComplete((res, e) -&gt; {
//操作完成时，唤醒在await()方法中等待的线程
l.countDown();
});
        boolean interrupted = false;
        while (!future.isDone()) {
            try {
                //阻塞等待
                l.await();
            } catch (InterruptedException e) {
                interrupted = true;
                break;
            }
        }

        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    }

    if (future.isSuccess()) {
        return future.getNow();
    }

    throw convertException(future);
}
```


org.redisson.RedissonLock#unlockInnerAsync解锁流程图：
![](https://i.imgur.com/vD8vYYG.jpg)


**实现源码：** 

```
protected RFuture unlockInnerAsync(long threadId) {
/**
* 通过 EVAL 命令执行 Lua 脚本获取锁，保证了原子性
*/
return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
//如果分布式锁存在，但是value不匹配，表示锁已经被其他线程占用，无权释放锁，那么直接返回空值（解铃还须系铃人）
"if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
"return nil;" +
"end; " +
//如果value匹配，则就是当前线程占有分布式锁，那么将重入次数减1
"local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
//重入次数减1后的值如果大于0，表示分布式锁有重入过，那么只能更新失效时间，还不能删除
"if (counter &gt; 0) then " +
"redis.call('pexpire', KEYS[1], ARGV[2]); " +
"return 0; " +
"else " +
//重入次数减1后的值如果为0，这时就可以删除这个KEY，并发布解锁消息，返回1
"redis.call('del', KEYS[1]); " +
"redis.call('publish', KEYS[2], ARGV[1]); " +
"return 1; "+
"end; " +
"return nil;",
//这5个参数分别对应KEYS[1]，KEYS[2]，ARGV[1]，ARGV[2]和ARGV[3]
Arrays.asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
```

**解锁消息处理**

```
org.redisson.pubsub#onMessage   

public class LockPubSub extends PublishSubscribe {
public static final Long UNLOCK_MESSAGE = 0L;
public static final Long READ_UNLOCK_MESSAGE = 1L;

public LockPubSub(PublishSubscribeService service) {
    super(service);
}

@Override
protected RedissonLockEntry createEntry(RPromise&lt;RedissonLockEntry&gt; newPromise) {
    return new RedissonLockEntry(newPromise);
}

@Override
protected void onMessage(RedissonLockEntry value, Long message) {

    /**
     * 判断是否是解锁消息
     */
    if (message.equals(UNLOCK_MESSAGE)) {
        Runnable runnableToExecute = value.getListeners().poll();
        if (runnableToExecute != null) {
            runnableToExecute.run();
        }

        /**
         * 释放一个信号量，唤醒等待的entry.getLatch().tryAcquire去再次尝试申请锁
         */
        value.getLatch().release();
    } else if (message.equals(READ_UNLOCK_MESSAGE)) {
        while (true) {
            /**
             * 如果还有其他Listeners回调，则也唤醒执行
             */
            Runnable runnableToExecute = value.getListeners().poll();
            if (runnableToExecute == null) {
                break;
            }
            runnableToExecute.run();
        }

        value.getLatch().release(value.getLatch().getQueueLength());
    }
}
}
```
**总结对比**
通过Redisson实现分布式可重入锁（实现二），比纯自己通过set key value px milliseconds nx +lua实现（实现一）的效果更好些，虽然基本原理都一样。
因为通过分析源码可知，RedissonLock是可重入的，并且考虑了失败重试，可以设置锁的最大等待时间。
另外在实现上也做了一些优化，减少了无效的锁申请，提升了资源的利用率。
需要特别注意的是，RedissonLock 同样没有解决节点挂掉的时候，存在丢失锁的风险的问题。
而现实情况是有一些场景无法容忍的，所以Redisson提供了实现了redlock算法的RedissonRedLock，RedissonRedLock真正解决了单点失败的问题，代价是需要额外的为 RedissonRedLock搭建Redis环境。
所以，如果业务场景可以容忍这种小概率的错误，则推荐使用 RedissonLock，如果无法容忍，则推荐使用 RedissonRedLock。

**Redlock算法**
在分布式版本的算法里我们假设我们有N个Redis master节点，这些节点都是完全独立的，我们不用任何复制或者其他隐含的分布式协调机制。
之前我们已经描述了在Redis单实例下怎么安全地获取和释放锁。我们确保将在每（N)个实例上使用此方法获取和释放锁。
在我们的例子里面我们把N设成5，这是一个比较合理的设置，所以我们需要在5台机器上面或者5台虚拟机上面运行这些实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:
* 获取当前Unix时间，以毫秒为单位。
* 依次尝试从5个实例，使用相同的key和具有唯一性的value（例如UUID）获取锁。当向Redis请求获取锁时，客户端应该设置一个尝试从某个Reids实例获取锁的最大等待时间（超过这个时间，则立马询问下一个实例），这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
* 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁消耗的时间。当且仅当从大多数（N/2+1，这里是3个节点）的Redis节点都取到锁，并且使用的总耗时小于锁失效时间时，锁才算获取成功。
* 如果取到了锁，key的真正有效时间 = 有效时间（获取锁时设置的key的自动超时时间） – 获取锁的总耗时（询问各个Redis实例的总耗时之和）（步骤3计算的结果）
* 如果因为某些原因，最终获取锁失败（即没有在至少 “N/2+1 ”个Redis实例取到锁或者“获取锁的总耗时”超过了“有效时间”），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功，这样可以防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）

### 4、用Redisson实现分布式锁(红锁RedissonRedLock)及源码分析（实现三）
这里以三个单机模式为例，需要特别注意的是他们完全互相独立，不存在主从复制或者其他集群协调机制。
```
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://172.0.0.1:5378").setPassword("a123456").setDatabase(0);
RedissonClient redissonClient1 = Redisson.create(config1);
Config config2 = new Config();
    config2.useSingleServer().setAddress("redis://172.0.0.1:5379").setPassword("a123456").setDatabase(0);
    RedissonClient redissonClient2 = Redisson.create(config2);

    Config config3 = new Config();
    config3.useSingleServer().setAddress("redis://172.0.0.1:5380").setPassword("a123456").setDatabase(0);
    RedissonClient redissonClient3 = Redisson.create(config3);

    /**
     * 获取多个 RLock 对象
     */
    RLock lock1 = redissonClient1.getLock(lockKey);
    RLock lock2 = redissonClient2.getLock(lockKey);
    RLock lock3 = redissonClient3.getLock(lockKey);

    /**
     * 根据多个 RLock 对象构建 RedissonRedLock （最核心的差别就在这里）
     */
    RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);

    try {
        /**
         * 4.尝试获取锁
         * waitTimeout 尝试获取锁的最大等待时间，超过这个值，则认为获取锁失败
         * leaseTime   锁的持有时间,超过这个时间锁会自动失效（值应设置为大于业务处理的时间，确保在锁有效期内业务能处理完）
         */
        boolean res = redLock.tryLock((long)waitTimeout, (long)leaseTime, TimeUnit.SECONDS);
        if (res) {
            //成功获得锁，在这里处理业务
        }
    } catch (Exception e) {
        throw new RuntimeException("aquire lock fail");
    }finally{
        //无论如何, 最后都要解锁
        redLock.unlock();
    }
```
最核心的变化就是需要构建多个RLock，然后根据多个RLock构建成一个RedissonRedLock。

因为redLock算法是建立在多个互相独立的Redis环境之上的（为了区分可以叫为Redission node），Redission node节点既可以是单机模式(single)，也可以是主从模式(master/salve)，哨兵模式(sentinal)，或者集群模式(cluster)。

这就意味着，不能跟以往这样只搭建1个cluster、或1个 sentinel集群，或是1套主从架构就了事了，需要为 RedissonRedLock额外搭建多几套独立的Redission节点。 

比如可以搭建3个或者5个Redission节点，具体可看视资源及业务情况而定。

下图是一个利用多个Redission node最终组成RedLock分布式锁的例子，需要特别注意的是每个Redission node是互相独立的，不存在任何复制或者其他隐含的分布式协调机制。
![](https://i.imgur.com/GcatGZR.png)

**Redisson实现redlock算法源码分析（RedLock）**
加锁核心代码org.redisson.RedissonMultiLock#tryLock
```
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
long newLeaseTime = -1;
if (leaseTime != -1) {
newLeaseTime = unit.toMillis(waitTime)*2;
}
    long time = System.currentTimeMillis();
    long remainTime = -1;
    if (waitTime != -1) {
        remainTime = unit.toMillis(waitTime);
    }
    long lockWaitTime = calcLockWaitTime(remainTime);
    /**
     * 1. 允许加锁失败节点个数限制（N-(N/2+1)）
     */
    int failedLocksLimit = failedLocksLimit();
    /**
     * 2. 遍历所有节点通过EVAL命令执行lua加锁
     */
    List&lt;RLock&gt; acquiredLocks = new ArrayList&lt;&gt;(locks.size());
    for (ListIterator&lt;RLock&gt; iterator = locks.listIterator(); iterator.hasNext();) {
        RLock lock = iterator.next();
        boolean lockAcquired;
        /**
         *  3.对节点尝试加锁
         */
        try {
            if (waitTime == -1 &amp;&amp; leaseTime == -1) {
                lockAcquired = lock.tryLock();
            } else {
                long awaitTime = Math.min(lockWaitTime, remainTime);
                lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
            }
        } catch (RedisResponseTimeoutException e) {
            // 如果抛出这类异常，为了防止加锁成功，但是响应失败，需要解锁所有节点
            unlockInner(Arrays.asList(lock));
            lockAcquired = false;
        } catch (Exception e) {
            // 抛出异常表示获取锁失败
            lockAcquired = false;
        }

        if (lockAcquired) {
            /**
             *4. 如果获取到锁则添加到已获取锁集合中
             */
            acquiredLocks.add(lock);
        } else {
            /**
             * 5. 计算已经申请锁失败的节点是否已经到达 允许加锁失败节点个数限制 （N-(N/2+1)）
             * 如果已经到达， 就认定最终申请锁失败，则没有必要继续从后面的节点申请了
             * 因为 Redlock 算法要求至少N/2+1 个节点都加锁成功，才算最终的锁申请成功
             */
            if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                break;
            }

            if (failedLocksLimit == 0) {
                unlockInner(acquiredLocks);
                if (waitTime == -1 &amp;&amp; leaseTime == -1) {
                    return false;
                }
                failedLocksLimit = failedLocksLimit();
                acquiredLocks.clear();
                // reset iterator
                while (iterator.hasPrevious()) {
                    iterator.previous();
                }
            } else {
                failedLocksLimit--;
            }
        }

        /**
         * 6.计算 目前从各个节点获取锁已经消耗的总时间，如果已经等于最大等待时间，则认定最终申请锁失败，返回false
         */
        if (remainTime != -1) {
            remainTime -= System.currentTimeMillis() - time;
            time = System.currentTimeMillis();
            if (remainTime &lt;= 0) {
                unlockInner(acquiredLocks);
                return false;
            }
        }
    }

    if (leaseTime != -1) {
        List&lt;RFuture&lt;Boolean&gt;&gt; futures = new ArrayList&lt;&gt;(acquiredLocks.size());
        for (RLock rLock : acquiredLocks) {
            RFuture&lt;Boolean&gt; future = ((RedissonLock) rLock).expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
            futures.add(future);
        }

        for (RFuture&lt;Boolean&gt; rFuture : futures) {
            rFuture.syncUninterruptibly();
        }
    }

    /**
     * 7.如果逻辑正常执行完则认为最终申请锁成功，返回true
     */
    return true;
}
```


## 十、Redis事务
* 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断；
* 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行；
* Redis事务相关的命令有

| 命令 | 功能 | 
| -------- | -------- | 
|multi|开启事务|
|exec|提交事务|
|discard|回滚事务|
|watch|监视key，如果key被改变，则打断事务执行|
|unwatch|取消监视|

**redis事务特征**：
每一条命令都具有独立的原子性，所以redis的事务特征仅仅是保证批量的提交或者放弃提交。

**执行过程**
* watch设置监视key，如果key被其他客户端改变，打断事务执行，exec返回为空数组。这个命令其实也要谨慎使用。
* multi命令用于开启一个事务，接收到的后续命令均不会被立即执行，而是放到一个队列中，等待执行。
* exec命令被调用时，事务队列中的命令将被逐个执行，但执行发生错误的时候，其他命令依旧会继续执行。此命令的返回值是一个数组，会返回每条命令的执行情况。
* discard命令百调用时，事务回滚，其实就是清空队列，放弃提交的操作。
* unwatch取消监视

**具体示例如下**
```
@Autowired
private RedisTemplate<String, String> redisTemplate;
 
 
@RequestMapping(value = "/api/v1/test", method = RequestMethod.GET)
public ResultBody query4largeCar() {
 
    String key1 = "key1";
    String key2 = "key2";
 
    // 测试事务提交 start
    try {
        // 设置开启事务支持
        redisTemplate.setEnableTransactionSupport(true);
        // 监视key事务中的key，如果key被其他客户端改变，打断事务执行，exec返回为空数组
        redisTemplate.watch(Arrays.asList(key1,key2));
        // 启动事务
        redisTemplate.multi();
 
        ValueOperations<String, String> operations = redisTemplate.opsForValue();
        // 插入缓存
        operations.set(key1,"333", 1, TimeUnit.HOURS);
        operations.set(key2,"333", 1, TimeUnit.HOURS);
 
        // 提交事务
        List<Object> objects = redisTemplate.exec();
        System.out.println(objects);
        // 事务执行完成，取消监视
        redisTemplate.unwatch();
 
        // 检查事务提交情况
        System.out.println("==========事务提交，88888代表成功========="+operations.get(key1));
        System.out.println("==========事务提交，88888代表成功========="+operations.get(key2));
    }catch (Exception e){
        e.printStackTrace();
    }
 
    // 测试事务回滚 start
    try {
        // 设置开启事务支持
        redisTemplate.setEnableTransactionSupport(true);
        // 监视key事务中的key，如果发生改变，那么表明事务失败，打断事务执行
        redisTemplate.watch(Arrays.asList(key1,key2));
        // 启动事务
        redisTemplate.multi();
 
 
        ValueOperations<String, String> operations = redisTemplate.opsForValue();
        // 插入缓存
        operations.set(key1,"xxxxx", 1, TimeUnit.HOURS);
        operations.set(key2,"xxxxx", 1, TimeUnit.HOURS);
 
        // 回滚事务
        redisTemplate.discard();
        // 事务执行完成，取消监视
        redisTemplate.unwatch();
 
        // 检查事务提交情况
        System.out.println("==========事务回滚，xxxxx代表失败========="+operations.get(key1));
        System.out.println("==========事务回滚，xxxxx代表失败========="+operations.get(key2));
    }catch (Exception e){
        e.printStackTrace();
    }
    return new ResultBody(ApiResultEnum.SUCCESS);
}
```

## 十一、Redis和Memcached的对比
两者都是非关系型内存键值数据库。有以下主要不同：
### 1、数据类型
Memcached仅支持字符串类型；
而Redis支持五种不同种类的数据类型，使得它可以更灵活地解决问题。
### 2、数据持久化
Memcached 不支持持久化；
Redis 支持两种持久化策略：RDB快照和AOF日志。
### 3、分布式
Memcached 不支持分布式，只能通过在客户端使用像一致性哈希这样的分布式算法来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。
Redis Cluster 实现了分布式的支持。
### 4、内存管理机制
Memcached 将内存分割成特定长度的块来存储数据，以完全解决内存碎片的问题，但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就浪费掉了。
在 Redis 中，并不是所有数据都一直存储在内存中，可以将一些很久没用的 value 交换到磁盘。而 Memcached 的数据则会一直在内存中。
 
 
## 十二、Redis应用场景
### （1）会话缓存（Session Cache）
最常用的一种使用Redis的情景是会话缓存（session cache）。用Redis缓存会话比其他存储（如Memcached）的优势在于：Redis提供持久化。当维护一个不是严格要求一致性的缓存时，如果用户的购物车信息全部丢失，大部分人都会不高兴的，现在，他们还会这样吗？
幸运的是，随着 Redis 这些年的改进，很容易找到怎么恰当的使用Redis来缓存会话的文档。甚至广为人知的商业平台Magento也提供Redis的插件。

### （2）全页缓存（FPC）
除基本的会话token之外，Redis还提供很简便的FPC平台。回到一致性问题，即使重启了Redis实例，因为有磁盘的持久化，用户也不会看到页面加载速度的下降，这是一个极大改进，类似PHP本地FPC。
再次以Magento为例，Magento提供一个插件来使用Redis作为全页缓存后端。
此外，对WordPress的用户来说，Pantheon有一个非常好的插件 wp-redis，这个插件能帮助你以最快速度加载你曾浏览过的页面。

### （3）队列
Reids在内存存储引擎领域的一大优点是提供list和set 操作，这使得Redis能作为一个很好的消息队列平台来使用。Redis作为队列使用的操作，就类似于本地程序语言对list的push/pop操作。一般使用list结构作为队列，rpush生产消息，lpop消费消息，当lpop没有消息的时候，要适当sleep一会再重试；list还有个指令叫blpop，在没有消息的时候，它会阻塞住直到消息到来，这样就不需要再没有消息的时候哦进行sleep。
**redis如何实现延时队列**
使用sortedset，拿时间戳作为score，消息内容作为key调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。

### （4）排行榜/计数器
Redis在内存中对数字进行递增或递减的操作实现的非常好。集合（Set）和有序集合（Sorted Set）也使得我们在执行这些操作的时候变的非常简单，Redis只是正好提供了这两种数据结构。
所以，我们要从排序集合中获取到排名最靠前的10个用户–我们称之为“user_scores”，我们只需要像下面一样执行即可：
当然，这是假定你是根据你用户的分数做递增的排序。如果你想返回用户及用户的分数，你需要这样执行：
ZRANGE user_scores 0 10 WITHSCORES
Agora Games就是一个很好的例子，用Ruby实现的，它的排行榜就是使用Redis来存储数据的，你可以在这里看到。

### （5）发布/订阅
最后（但肯定不是最不重要的）是Redis的发布/订阅功能。发布/订阅的使用场景确实非常多。我已看见人们在社交网络连接中使用，还可作为基于发布/订阅的脚本触发器，甚至用Redis的发布/订阅功能来建立聊天系统！

## 十三、redis常见问题处理
* 缓存雪崩
* 缓存穿透
* 缓存与数据库双写一致

### 1、缓存雪崩
#### 1.1、什么是缓存雪崩？
缓存(Redis)的作用：

* 提高性能，缓存查询速度比数据库查询速度快
* 提高并发性能，请求分流，提高并发

但是Redis不可能把所有的数据都缓存起来(内存昂贵且有限)，所以Redis需要对数据设置过期时间，并采用的是惰性删除 + 定期删除两种策略对过期数据进行清理。

如果缓存数据设置的过期时间是相同的，并且Redis恰好将这部分数据全部删光了。这就会导致在这段时间内，这些缓存同时失效，全部请求到数据库中发生缓存雪崩。缓存雪崩如果发生了，很可能就把我们的数据库搞垮，导致整个服务瘫痪！

#### 1.2、如何解决缓存雪崩？
对于“对缓存数据设置相同的过期时间，导致某段时间内缓存失效，请求全部走数据库。”这种情况，解决方法：在缓存的时候给过期时间加上一个随机值，这样就会大幅度的减少缓存在同一时间过期。

对于“Redis挂掉了，请求全部走数据库”这种情况，我们可以有以下的思路：
* 事发前：实现Redis的高可用(主从架构+Sentinel 或者Redis Cluster)，尽量避免Redis挂掉这种情况发生。
* 事发中：万一Redis真的挂了，我们可以设置本地缓存(ehcache)+限流(hystrix)，尽量避免我们的数据库被干掉(起码能保证我们的服务还是能正常工作的)
* 事发后：redis持久化，重启后自动从磁盘上加载数据，快速恢复缓存数据。

### 2、缓存穿透
#### 2.1、什么是缓存穿透
缓存穿透是指查询一个一定不存在的数据。由于缓存不命中，并且出于容错考虑，如果从数据库查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，失去了缓存的意义。简单来说就是请求的数据在缓存大量不命中，导致请求走数据库。缓存穿透如果发生了，也可能把我们的数据库搞垮，导致整个服务瘫痪！

#### 2.2、如何解决缓存穿透
解决缓存穿透也有两种方案：
* 由于请求的参数是不合法的(每次都请求不存在的参数)，于是我们可以使用布隆过滤器(BloomFilter)或者压缩filter提前拦截，不合法就不让这个请求到数据库层！
* 当我们从数据库找不到的时候，我们也将这个空对象设置到缓存里边去。下次再请求的时候，就可以从缓存里边获取了(这种情况我们一般会将空对象设置一个较短的过期时间)。

### 3、缓存与数据库双写一致
#### 3.1、对于读操作，流程是这样的
上面讲缓存穿透的时候也提到了：如果从数据库查不到数据则不写入缓存。一般我们对读操作的时候有这么一个固定的套路：
* 如果我们的数据在缓存里边有，那么就直接取缓存的。
* 如果缓存里没有我们想要的数据，我们会先去查询数据库，然后将数据库查出来的数据写到缓存中。
* 最后将数据返回给请求

#### 3.2、什么是缓存与数据库双写一致问题？
如果仅仅查询的话，缓存的数据和数据库的数据是没问题的。但是，当我们要更新的时候呢？各种情况很可能就造成数据库和缓存的数据不一致了。
从理论上说，只要我们设置了键的过期时间，我们就能保证缓存和数据库的数据最终是一致的。因为只要缓存数据过期了，就会被删除。随后读的时候，因为缓存里没有，就可以查数据库的数据，然后将数据库查出来的数据写入到缓存中。
除了设置过期时间，我们还需要做更多的措施来尽量避免数据库与缓存处于不一致的情况发生。

#### 3.3、对于更新操作
一般来说，执行更新操作时，我们会有两种选择：
* 先操作数据库，再操作缓存
* 先操作缓存，再操作数据库

首先，要明确的是，无论我们选择哪个，我们都希望这两个操作要么同时成功，要么同时失败。所以，这会演变成一个分布式事务的问题。所以，如果原子性被破坏了，可能会有以下的情况：
* 操作数据库成功了，操作缓存失败了。
* 操作缓存成功了，操作数据库失败了。

如果第一步已经失败了，我们直接返回Exception出去就好了，第二步根本不会执行。

下面我们具体来分析一下吧。

**操作缓存**
操作缓存也有两种方案：
* 更新缓存
* 删除缓存

一般我们都是采取删除缓存缓存策略的，原因如下：
* 如果加上更新缓存，那就更加容易导致数据库与缓存数据不一致问题(删除缓存直接和简单很多)；
* 如果每次更新了数据库，都要更新缓存，不如直接删除掉等再次读取时，缓存里没有，再到数据库找，数据库中找到就写入缓存（懒加载）。

**先更新数据库，再删除缓存**
正常的情况是这样的：先操作数据库，成功；再删除缓存，也成功。
如果原子性被破坏了：第一步成功(操作数据库)，第二步失败(删除缓存)，会导致数据库里是新数据，而缓存里是旧数据；如果第一步(操作数据库)就失败了，我们可以直接返回错误(Exception)，不会出现数据不一致；如果在高并发的场景下，出现数据库与缓存数据不一致的概率特别低，也不是没有：
* 缓存刚好失效
* 线程A查询数据库，得一个旧值
* 线程B将新值写入数据库
* 线程B删除缓存
* 线程A将查到的旧值写入缓存

要达成上述情况，还是说一句概率特别低

因为这个条件需要发生在读缓存时缓存失效，而且并发着有一个写操作。而实际上数据库的写操作会比读操作慢得多，而且还要锁表，而读操作必需在写操作前进入数据库操作，而又要晚于写操作更新缓存，所有的这些条件都具备的概率基本并不大。

对于这种策略，其实是一种设计模式：Cache Aside Pattern
删除缓存失败的解决思路：
* 将需要删除的key发送到消息队列中
* 自己消费消息，获得需要删除的key
* 不断重试删除操作，直到成功

**先删除缓存，再更新数据库**
正常情况是这样的：先删除缓存，成功；再更新数据库，也成功；
如果原子性被破坏了：第一步成功(删除缓存)，第二步失败(更新数据库)，数据库和缓存的数据还是一致的；如果第一步(删除缓存)就失败了，我们可以直接返回错误(Exception)，数据库和缓存的数据还是一致的。看起来是很美好，但是我们在并发场景下分析一下，就知道还是有问题的了：
* 线程A删除了缓存
* 线程B查询，发现缓存已不存在
* 线程B去数据库查询得到旧值
* 线程B将旧值写入缓存
* 线程A将新值写入数据库

所以也会导致数据库和缓存不一致的问题。
并发下解决数据库与缓存不一致的思路：将删除缓存、修改数据库、读取缓存等的操作积压到队列里边，实现串行化。

**对比两种策略**
我们可以发现，两种策略各自有优缺点：
* 先删除缓存，再更新数据库；在高并发下表现不如意，在原子性被破坏时表现优异
* 先更新数据库，再删除缓存(Cache Aside Pattern设计模式)；在高并发下表现优异，在原子性被破坏时表现不如意

**其他保障数据一致的方案**
可以用databus或者阿里的canal监听binlog进行更新。

## 十四、Lua脚本
Lua 脚本功能是Reids 2.6版本的最大亮点，通过内嵌对Lua环境的支持，Redis解决了长久以来不能高效地处理CAS（check-and-set）命令的缺点，并且可以通过组合使用多个命令，轻松实现以前很难实现或者不能高效实现的模式。

先介绍Lua环境的初始化步骤，然后对Lua脚本的安全性问题、以及解决这些问题的方法进行说明，最后对执行Lua脚本的两个命令——EVAL和EVALSHA的实现原理进行介绍。

### 1、初始化 Lua 环境
在初始化Redis服务器时，对Lua环境的初始化也会一并进行。为了让Lua环境符合Redis脚本功能的需求，Redis对Lua环境进行了一系列的修改，包括添加函数库、更换随机函数、保护全局变量等等。

**整个初始化Lua环境的步骤如下：**
（1）调用lua_open函数，创建一个新的Lua环境。
（2）载入指定的Lua函数库，包括：
* 基础库（base lib）。
* 表格库（table lib）。
* 字符串库（string lib）。
* 数学库（math lib）。
* 调试库（debug lib）。
* 用于处理JSON对象的cjson库。在Lua值和C结构（struct）之间进行转换的struct库（http://www.inf.puc-rio.br/~roberto/struct/）。
* 处理MessagePack数据的cmsgpack（https://github.com/antirez/lua-cmsgpack）。

（3）屏蔽一些可能对Lua环境产生安全问题的函数，比如 loadfile。
（4）创建一个Redis字典，保存Lua脚本，并在复制（replication）脚本时使用。字典的键为SHA1校验和，字典的值为 Lua 脚本。
（5） 创建一个 redis 全局表格到Lua环境，表格中包含了各种对 Redis 进行操作的函数，包括：
* 用于执行Redis命令的redis.call和redis.pcall函数。
* 用于发送日志的redis.log函数，以及相应的日志级别：
*  redis.LOG_DEBUG、redis.LOG_VERBOSE、redis.LOG_NOTICE、redis.LOG_WARNING 
* 用于计算SHA1校验和的redis.sha1hex函数。
* 用于返回错误信息的redis.error_reply函数和redis.status_reply函数。

（6）用Redis自己定义的随机生成函数，替换math表原有的math.random函数和math.randomseed函数，新的函数具有这样的性质：每次执行Lua脚本时，除非显式地调用 math.randomseed，否则math.random生成的伪随机数序列总是相同的。
（7）创建一个对Redis多批量回复（multi bulk reply）进行排序的辅助函数。
（8）对Lua环境中的全局变量进行保护，以免被传入的脚本修改。
（9）因为Redis命令必须通过客户端来执行，所以需要在服务器状态中创建一个无网络连接的伪客户端（fake client），专门用于执行 Lua 脚本中包含的Redis命令：当Lua脚本需要执行 Redis 命令时，它通过伪客户端来向服务器发送命令请求，服务器在执行完命令之后，将结果返回给伪客户端，而伪客户端又转而将命令结果返回给Lua脚本。
（10）将Lua环境的指针记录到Redis服务器的全局状态中，等候Redis的调用。
以上就是Redis初始化Lua环境的整个过程， 当这些步骤都执行完之后，Redis就可以使用Lua环境来处理脚本了。

严格来说， 步骤1至8才是初始化Lua环境的操作， 而步骤9和10 则是将Lua环境关联到服务器的操作，为了按顺序观察整个初始化过程， 我们将两种操作放在了一起。
另外，步骤6用于创建无副作用的脚本，而步骤7则用于去除部分Redis命令中的不确定性（non deterministic），关于这两点，请看下面一节关于脚本安全性的讨论。

### 2、脚本的安全性
当将Lua脚本复制到附属节点， 或者将Lua脚本写入AOF文件时，Redis需要解决这样一个问题：如果一段Lua脚本带有随机性质或副作用，那么当这段脚本在附属节点运行时，或者从AOF文件载入重新运行时，它得到的结果可能和之前运行的结果完全不同。

考虑以下一段代码，其中的get_random_number()带有随机性质，我们在服务器SERVER中执行这段代码，并将随机数的结果保存到键number上：

```
# 虚构例子，不会真的出现在脚本环境中
redis> EVAL "return redis.call('set', KEYS[1], get_random_number())" 1 number
OK

redis> GET number
"10086"
```
现在，假如EVAL的代码被复制到了附属节点SLAVE，因为 get_random_number()的随机性质，它有很大可能会生成一个和 10086完全不同的值，比如65535：

```
# 虚构例子，不会真的出现在脚本环境中

redis> EVAL "return redis.call('set', KEYS[1], get_random_number())" 1 number
OK

redis> GET number
"65535"
```
可以看到，带有随机性的写入脚本产生了一个严重的问题： 它破坏了服务器和附属节点数据之间的一致性。
当从AOF文件中载入带有随机性质的写入脚本时，也会发生同样的问题。
只有在带有随机性的脚本进行写入时，随机性才是有害的。
如果一个脚本只是执行只读操作，那么随机性是无害的。

比如说，如果脚本只是单纯地执行RANDOMKEY命令，那么它是无害的；但如果在执行RANDOMKEY之后，基于RANDOMKEY的结果进行写入操作，那么这个脚本就是有害的。

和随机性质类似， 如果一个脚本的执行对任何副作用产生了依赖， 那么这个脚本每次执行所产生的结果都可能会不一样。

为了解决这个问题， Redis对Lua环境所能执行的脚本做了一个严格的限制——所有脚本都必须是无副作用的纯函数（pure function）。

为此，Redis对Lua环境做了一些列相应的措施：
* 不提供访问系统状态状态的库（比如系统时间库）。
* 禁止使用loadfile函数。
* 如果脚本在执行带有随机性质的命令（比如RANDOMKEY），或者带有副作用的命令（比如TIME）之后，试图执行一个写入命令（比如SET），那么Redis将阻止这个脚本继续运行，并返回一个错误。
* 如果脚本执行了带有随机性质的读命令（比如 SMEMBERS ），那么在脚本的输出返回给 Redis 之前，会先被执行一个自动的字典序排序，从而确保输出结果是有序的。
* 用Redis自己定义的随机生成函数，替换Lua环境中math表原有的math.random函数和math.randomseed函数，新的函数具有这样的性质：每次执行Lua脚本时，除非显式地调用 math.randomseed ，否则math.random生成的伪随机数序列总是相同的。

经过这一系列的调整之后，Redis可以保证被执行的脚本：
* 无副作用。
* 没有有害的随机性。
* 对于同样的输入参数和数据集，总是产生相同的写入命令。

### 3、脚本的执行
在脚本环境的初始化工作完成以后，redis就可以通过EVAL命令或EVALSHA命令执行Lua脚本了。
其中，EVAL直接对输入的脚本代码体（body）进行求值：
```
redis> EVAL "return 'hello world'" 0
"hello world"
```
而EVALSHA则要求输入某个脚本的SHA1校验和，这个校验和所对应的脚本必须至少被 EVAL 执行过一次：
```
redis> EVAL "return 'hello world'" 0
"hello world"

redis> EVALSHA 5332031c6b470dc5a0dd9b4bf2030dea6d65de91 0    // 上一个脚本的校验和
"hello world"
```
或者曾经使用SCRIPT LOAD载入过这个脚本：
```
redis> SCRIPT LOAD "return 'dlrow olleh'"
"d569c48906b1f4fca0469ba4eee89149b5148092"

redis> EVALSHA d569c48906b1f4fca0469ba4eee89149b5148092 0
"dlrow olleh"
```
因为EVALSHA是基于EVAL构建的， 所以先讲解EVAL的实现之后再讲解EVALSHA的实现。

### 4、EVAL命令的实现
EVAL命令的执行可以分为以下步骤：
* 为输入脚本定义一个Lua函数。
* 执行这个Lua函数。

以下两个小节分别介绍这两个步骤。

#### （1）定义Lua函数
所有被Redis执行的Lua脚本， 在Lua环境中都会有一个和该脚本相对应的无参数函数： 当调用EVAL命令执行脚本时， 程序第一步要完成的工作就是为传入的脚本创建一个相应的Lua函数。
举个例子，当执行命令EVAL "return 'hello world'" 0 时，Lua会为脚本 "return 'hello world'"创建以下函数：

```
function f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91()
    return 'hello world'
end
```
其中，函数名以f_为前缀， 后跟脚本的SHA校验和（一个 40 个字符长的字符串）拼接而成。 而函数体（body）则是用户输入的脚本。

**以函数为单位保存Lua脚本有以下好处：**
* 执行脚本的步骤非常简单，只要调用和脚本相对应的函数即可。
* Lua环境可以保持清洁，已有的脚本和新加入的脚本不会互相干扰，也可以将重置Lua环境和调用Lua GC的次数降到最低。
* 如果某个脚本所对应的函数在Lua环境中被定义过至少一次，那么只要记得这个脚本的SHA1校验和，就可以直接执行该脚本——这是实现EVALSHA命令的基础，稍后在介绍EVALSHA的时候就会说到这一点。
* 在为脚本创建函数前，程序会先用函数名检查 Lua 环境，只有在函数定义未存在时，程序才创建函数。重复定义函数一般并没有什么副作用，这算是一个小优化。
* 另外，如果定义的函数在编译过程中出错（比如，脚本的代码语法有错），那么程序向用户返回一个脚本错误，不再执行后面的步骤。
 
#### （2）执行Lua函数
在定义好Lua函数之后，程序就可以通过运行这个函数来达到运行输入脚本的目的了。不过，在此之前为了确保脚本的正确和安全执行，还需要执行一些设置钩子、传入参数之类的操作，整个**执行函数的过程**如下：
* 将EVAL命令中输入的KEYS参数和ARGV参数以全局数组的方式传入到Lua环境中。
* 设置伪客户端的目标数据库为调用者客户端的目标数据库： fake_client->db = caller_client->db ，确保脚本中执行的Redis命令访问的是正确的数据库。
* 为Lua环境装载超时钩子，保证在脚本执行出现超时时可以杀死脚本，或者停止Redis服务器。
* 执行脚本对应的Lua函数。
* 如果被执行的Lua脚本中带有SELECT命令，那么在脚本执行完毕之后，伪客户端中的数据库可能已经有所改变，所以需要对调用者客户端的目标数据库进行更新： caller_client->db = fake_client->db 。
* 执行清理操作：清除钩子；清除指向调用者客户端的指针；等等。
* 将 Lua 函数执行所得的结果转换成 Redis 回复，然后传给调用者客户端。
* 对 Lua 环境进行一次单步的渐进式 GC 。

以下是执行 EVAL "return 'hello world'" 0 的过程中， 调用者客户端（caller）、Redis 服务器和 Lua 环境之间的数据流表示图：

```
          发送命令请求
          EVAL "return 'hello world'" 0
Caller ----------------------------------------> Redis

          为脚本 "return 'hello world'"
          创建 Lua 函数
Redis  ----------------------------------------> Lua

          绑定超时处理钩子
Redis  ----------------------------------------> Lua

          执行脚本函数
Redis  ----------------------------------------> Lua

          返回函数执行结果（一个 Lua 值）
Redis  <---------------------------------------- Lua

          将 Lua 值转换为 Redis 回复
          并将结果返回给客户端
Caller <---------------------------------------- Redis
```
上面这个图可以作为所有Lua脚本的基本执行流程图，不过它展示的Lua脚本中不带有Redis命令调用：当Lua脚本里本身有调用 Redis命令时（执行redis.call或者redis.pcall ），Redis和Lua脚本之间的数据交互会更复杂一些。

举个例子，以下是执行命令EVAL "return redis.call('DBSIZE')" 0时，调用者客户端（caller）、伪客户端（fake client）、Redis服务器和Lua环境之间的数据流表示图：

```
          发送命令请求
          EVAL "return redis.call('DBSIZE')" 0
Caller ------------------------------------------> Redis

          为脚本 "return redis.call('DBSIZE')"
          创建 Lua 函数
Redis  ------------------------------------------> Lua

          绑定超时处理钩子
Redis  ------------------------------------------> Lua

          执行脚本函数
Redis  ------------------------------------------> Lua

               执行 redis.call('DBSIZE')
Fake Client <------------------------------------- Lua

               伪客户端向服务器发送
               DBSIZE 命令请求
Fake Client -------------------------------------> Redis

               服务器将 DBSIZE 的结果
               （Redis 回复）返回给伪客户端
Fake Client <------------------------------------- Redis

               将命令回复转换为 Lua 值
               并返回给 Lua 环境
Fake Client -------------------------------------> Lua

          返回函数执行结果（一个 Lua 值）
Redis  <------------------------------------------ Lua

          将 Lua 值转换为 Redis 回复
          并将该回复返回给客户端
Caller <------------------------------------------ Redis
```
因为EVAL "return redis.call('DBSIZE')"只是简单地调用了一次DBSIZE命令， 所以Lua和伪客户端只进行了一趟交互，当脚本中的redis.call或者redis.pcall次数增多时，Lua和伪客户端的交互趟数也会相应地增多，不过总体的交互方法和上图展示的一样。

### 5、EVALSHA命令的实现
前面介绍EVAL命令的实现时说过，每个被执行过的Lua脚本，在Lua环境中都有一个和它相对应的函数，函数的名字由f_前缀加上40个字符长的SHA1校验和构成：比如f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91 。

只要脚本所对应的函数曾经在Lua里面定义过，那么即使用户不知道脚本的内容本身，也可以直接通过脚本的SHA1校验和来调用脚本所对应的函数，从而达到执行脚本的目的——这就是EVALSHA命令的实现原理。

可以用伪代码来描述这一原理：

```
def EVALSHA(sha1):

    # 拼接出 Lua 函数名字
    func_name = "f_" + sha1

    # 查看该函数是否已经在 Lua 中定义
    if function_defined_in_lua(func_name):

        # 如果已经定义过的话，执行函数
        return exec_lua_function(func_name)

    else:

        # 没有找到和输入 SHA1 值相对应的函数则返回一个脚本未找到错误
        return script_error("SCRIPT NOT FOUND")
```
除了执行 EVAL 命令之外， SCRIPT LOAD 命令也可以为脚本在 Lua 环境中创建函数：
```
redis> SCRIPT LOAD "return 'hello world'"
"5332031c6b470dc5a0dd9b4bf2030dea6d65de91"

redis> EVALSHA 5332031c6b470dc5a0dd9b4bf2030dea6d65de91 0
"hello world"
```
SCRIPT LOAD 执行的操作和前面《定义Lua函数》小节描述的一样。

### 6、错误传递
redis.call函数调用会产生错误，脚本遇到这种错误会返回怎样的信息呢？我们再看个例子
```
127.0.0.1:6379> hset foo x 1 y 2
(integer) 2
127.0.0.1:6379> eval 'return redis.call("incr", "foo")' 0
(error) ERR Error running script (call to f_8727c9c34a61783916ca488b366c475cb3a446cc): @user_script:1: WRONGTYPE Operation against a key holding the wrong kind of value
```
复制代码客户端输出的依然是一个通用的错误消息，而不是incr调用本应该返回的WRONGTYPE类型的错误消息。Redis内部在处理 redis.call遇到错误时是向上抛出异常，外围的用户看不见的 pcall调用捕获到脚本异常时会向客户端回复通用的错误信息。如果我们将上面的call改成pcall，结果就会不一样，它可以将内部指令返回的特定错误向上传递。
```
127.0.0.1:6379> eval 'return redis.pcall("incr", "foo")' 0
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```
### 7、脚本死循环怎么办？
Redis的指令执行是个单线程，这个单线程还要执行来自客户端的 lua 脚本。如果lua脚本中来一个死循环，是不是Redis就完蛋了？Redis为了解决这个问题，它提供了script kill指令用于动态杀死一个执行时间超时的lua脚本。不过script kill的执行有一个重要的前提，那就是当前正在执行的脚本没有对Redis的内部数据状态进行修改，因为Redis不允许script kill破坏脚本执行的原子性。比如脚本内部使用了redis.call("set", key, value) 修改了内部的数据，那么script kill执行时服务器会返回错误。下面我们来尝试以下script kill指令。
```
127.0.0.1:6379> eval 'while(true) do print("hello") end' 0
```
eval指令执行后，可以明显看出来redis卡死了，死活没有任何响应，如果去观察Redis服务器日志可以看到日志在疯狂输出hello字符串。这时候就必须重新开启一个redis-cli来执行 script kill指令。
```
127.0.0.1:6379> script kill
OK
(2.58s)
```
再回过头看eval指令的输出
```
127.0.0.1:6379> eval 'while(true) do print("hello") end' 0
(error) ERR Error running script (call to f_d395649372f578b1a0d3a1dc1b2389717cadf403): @user_script:1: Script killed by user with SCRIPT KILL...
(6.99s)
```
看到这里细心的同学会注意到两个疑点，第一个是script kill 指令为什么执行了2.58秒，第二个是脚本都卡死了，Redis哪里来的闲功夫接受script kill指令。如果你自己尝试了在第二个窗口执行redis-cli去连接服务器，你还会发现第三个疑点redis-cli建立连接有点慢，大约顿了有1秒左右。

### 8、Script Kill的原理
下面我就要开始揭秘kill的原理了，lua脚本引擎功能太强大了，它提供了各式各样的钩子函数，它允许在内部虚拟机执行指令时运行钩子代码。比如每执行N条指令执行一次某个钩子函数，Redis正是使用了这个钩子函数。

![](https://i.imgur.com/s0RMmyL.png)

```
void evalGenericCommand(client *c, int evalsha) {
  ...
  // lua引擎每执行10w条指令，执行一次钩子函数 luaMaskCountHook
  lua_sethook(lua,luaMaskCountHook,LUA_MASKCOUNT,100000);
  ...
}
```
Redis在钩子函数里会忙里偷闲去处理客户端的请求，并且只有在发现lua脚本执行超时之后才会去处理请求，这个超时时间默认是5秒。


### 9、小结
初始化 Lua 脚本环境需要一系列步骤，其中最重要的包括：
创建 Lua 环境。
载入 Lua 库，比如字符串库、数学库、表格库，等等。
创建 redis 全局表格，包含各种对 Redis 进行操作的函数，比如 redis.call 和 redis.log ，等等。
创建一个无网络连接的伪客户端，专门用于执行 Lua 脚本中的 Redis 命令。
Reids 通过一系列措施保证被执行的 Lua 脚本无副作用，也没有有害的写随机性：对于同样的输入参数和数据集，总是产生相同的写入命令。
EVAL 命令为输入脚本定义一个 Lua 函数，然后通过执行这个函数来执行脚本。
EVALSHA 通过构建函数名，直接调用 Lua 中已定义的函数，从而执行相应的脚本。