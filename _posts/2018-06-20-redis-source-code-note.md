---
title: redis源代码分析笔记
categories:
 - 源码分析
tags:
 - 源码分析
 - redis
---

## redis单进程单线程
main() 启动绑定端口，使用epoll事件机制，当有新的连接来的时候，使用acceptTcpHandler函数接收事件，如果是新的连接则新建一个socket接收，创建一个redisClient结构体，并新建一个用readQueryFromClient来处理的事件。

在新的事件的readQueryFromClient中，从socket中读取客户端上传的指令，解析指令，调用call(c,REDIS_CALL_FULL)函数执行命令获得返回结果，调用addReply()函数新建一个用sendReplyToClient来处理的事件。

在sendReplyToClient中处把结果返回给客户端。

事件控制是使用如epoll的epoll_ctl(state->epfd,op,fd,&ee)函数来控制事件产生时



## 数据结构

### zmalloc.c zmalloc分配内存时候，会在前面加一个大小的前缀。

### redis的字符串，记录字符串的长度和空闲长度。

```
struct sdshdr {
   int len;
   int free;
   char buf[];
};
```
长度为 0 的数组即 char buf[] 不占用内存

### redis的object，数据在ptr指针中，具体类型在type中存储。

```
typedef struct redisObject {
    // 刚刚好32 bits
    // 对象的类型，字符串/列表/集合/哈希表
    unsigned type:4;
    // 未使用的两个位
    unsigned notused:2; /* Not used */
    // 编码的方式，Redis 为了节省空间，提供多种方式来保存一个数据
    // 譬如：“123456789” 会被存储为整数123456789
    unsigned encoding:4;
    // 当内存紧张，淘汰数据的时候用到
    unsigned lru:22; /* lru time (relative to server.lruclock) */
    // 引用计数，引用数为0时候销毁，不同类型销毁方式不一样
    int refcount;
    // 数据指针
    void *ptr;
} robj;
```

### redis的哈希表

```
// 可以把它认为是一个链表，提示，开链法
typedef struct dictEntry {
    void *key;
    union {
    \\ val 指针可以指向一个redisObject
    void *val;
    uint64_t u64;
    int64_t s64;
    } v;
    struct dictEntry *next;
} dictEntry;
    // 要存储多种多样的数据结构，势必不同的数据有不同的哈希算法，不同的键值比较算法，
    // 不同的析构函数。
typedef struct dictType {
    // 哈希函数
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    // 比较函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 键值析构函数
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
    } dictType;
    // 一般哈希表数据结构
    /* This is our hash table structure. Every dictionary has two of this as we
    * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    // 两个哈希表
    dictEntry **table;
    // 哈希表的大小
    unsigned long size;
    // 哈希表大小掩码
    unsigned long sizemask;
    // 哈希表中数据项数量
    unsigned long used;
    } dictht;
    // 哈希表（字典）数据结构，Redis 的所有键值对都会存储在这里。其中包含两个哈希表。
typedef struct dict {
    // 哈希表的类型，包括哈希函数，比较函数，键值的内存释放函数
    dictType *type;
    // 存储一些额外的数据
    void *privdata;
    // 两个哈希表，默认是使用ht[0]，如果空间不够的话
    dictht ht[2];
    // 哈希表重置下标，指定的是哈希数组的数组下标
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 绑定到哈希表的迭代器个数
    int iterators; /* number of iterators currently running */
} dict;
```

### ziplist压缩链表：连续内存存储
ziplist 连续存储，提高内存的使用率，可存储字符串和整数，进cpu缓存，每次插入都要重新分配内存。set使用的ziplist来实现

```
<zlbytes><zltail><zllen><entry><entry><zlend>

typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;
```

### skiplist 跳表链表：堆内存动态分配存储，记录每层span。
skiplist，跳表用来实现zset。节点数据存储在robj中，分层level有多层记录下一个节点以及中间有多少步。查找时先从顶层的查，如果在区间内则到下一层的区间里面查。


```
// 跳表节点结构体
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    // 节点数据
    robj *obj;
    // 分数，游戏分数？按游戏分数排序
    double score;
    // 后驱指针
    struct zskiplistNode *backward;
    // 前驱指针数组TODO
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        // 调到下一个数据项需要走多少步，这在计算rank 的非常有帮助
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 跳表头尾指针
    struct zskiplistNode *header, *tail;
    // 跳表的长度
    unsigned long length;
    // 跳表的高度
    int level;
} zskiplist;
```

### intset整数集
intset 和 dict 都是 sadd 命令的底层数据结构，当添加的所有数据都是整数时，会使用前者；否则使用后者。特别的，当遇到添加数据为字符串，即不能表示为整数时，Redis 会把数据结构转换为 dict，即把 intset 中的数据全部搬迁到 dict。
intset中得数据会自动排序，查找时会使用二分法排序。

```
typedef struct intset {
// 每个整数的类型
uint32_t encoding;
// intset 长度
uint32_t length;
// 整数数组
int8_t contents[];
} intset;
```

## redis数据淘汰机制：

- volatile-lru(过期最近最少使用) 
- allkeys-lru(所有Key最近最少使用) 
- allkeys-random (所有key随机)
- volatile-ttl(过期最长事件)
- volatile-random(过期随机)

在主线程中

```
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
```

会不停的处理eventLoop中得文件事件FileEvent和定时事件TimeEvent。


## redis持久化rdb全量备份到文件dump.db
serverCron定时程序中rdbSaveBackground，在子进程中rdbSave中调用rdbSaveRio函数。

## redis持久化aof增量备份到文件
serverCron定时程序中rewriteAppendOnlyFileBackground，在子进程中rewriteAppendOnlyFile(tmpfile)中执行。


## redis订阅subscribe和发布publish

redis的redisServer中和redisClient中都有以下变量：  

```
    dict *pubsub_channels;  /* Map channels to list of subscribed clients */
    list *pubsub_patterns;  /* A list of pubsub_patterns */
```

当有client执行订阅subscribe某个对象时候调用pubsubSubscribeChannel函数中会将channel加入到redisClient和redisServer的pubsub_channels中。
代码如：dictAdd(c->pubsub_channels,channel,NULL)  dictAdd(server.pubsub_channels,channel,clients);

当执行发布时public时调用pubsubPublishMessage(robj *channel, robj *message)函数，该函数中会查找redisServer中channel dictFind(server.pubsub_channels,channel)，然后遍历下面的列表while ((ln = listNext(&li)) != NULL)对每一个client发送信息

```
        while ((ln = listNext(&li)) != NULL) {
            redisClient *c = ln->value;

            addReply(c,shared.mbulkhdr[3]);
            addReply(c,shared.messagebulk);
            addReplyBulk(c,channel);
            addReplyBulk(c,message);
            receivers++;
        }
```


## redis主从复制

redisServer中和主从复制相关的参数如下所示

```
struct redisServer {
    ...
    /* Replication (master) */
    int slaveseldb;                 /* Last SELECTed DB in replication output */
    long long master_repl_offset;   /* Global replication offset */
    int repl_ping_slave_period;     /* Master pings the slave every N seconds */
    char *repl_backlog;             /* Replication backlog for partial syncs 存储命令日志 */
    long long repl_backlog_size;    /* Backlog circular buffer size */
    long long repl_backlog_histlen; /* Backlog actual data length */
    long long repl_backlog_idx;     /* Backlog circular buffer current offset */
    long long repl_backlog_off;     /* Replication offset of first byte in the
                                       backlog buffer. */
    time_t repl_backlog_time_limit; /* Time without slaves after the backlog
                                       gets released. */
    time_t repl_no_slaves_since;    /* We have no slaves since that time.
                                       Only valid if server.slaves len is 0. */
    int repl_min_slaves_to_write;   /* Min number of slaves to write. */
    int repl_min_slaves_max_lag;    /* Max lag of <count> slaves to write. */
    int repl_good_slaves_count;     /* Number of slaves with lag <= max_lag. */
    int repl_diskless_sync;         /* Send RDB to slaves sockets directly. */
    int repl_diskless_sync_delay;   /* Delay to start a diskless repl BGSAVE. */
    /* Replication (slave) */
    char *masterauth;               /* AUTH with this password with master */
    char *masterhost;               /* Hostname of master */
    int masterport;                 /* Port of master */
    int repl_timeout;               /* Timeout after N seconds of master idle */
    redisClient *master;     /* Client that is master for this slave */
    redisClient *cached_master; /* Cached master to be reused for PSYNC. */
    int repl_syncio_timeout; /* Timeout for synchronous I/O calls */
    int repl_state;          /* Replication status if the instance is a slave */
    off_t repl_transfer_size; /* Size of RDB to read from master during sync. */
    off_t repl_transfer_read; /* Amount of RDB read from master during sync. */
    off_t repl_transfer_last_fsync_off; /* Offset when we fsync-ed last time. */
    int repl_transfer_s;     /* Slave -> Master SYNC socket */
    int repl_transfer_fd;    /* Slave -> Master SYNC temp file descriptor */
    char *repl_transfer_tmpfile; /* Slave-> master SYNC temp file name */
    time_t repl_transfer_lastio; /* Unix time of the latest read, for timeout */
    int repl_serve_stale_data; /* Serve stale data when link is down? */
    int repl_slave_ro;          /* Slave is read only? */
    time_t repl_down_since; /* Unix time at which link with master went down */
    int repl_disable_tcp_nodelay;   /* Disable TCP_NODELAY after SYNC? */
    int slave_priority;             /* Reported in INFO and used by Sentinel. */
    char repl_master_runid[REDIS_RUN_ID_SIZE+1];  /* Master run id for PSYNC. */
    long long repl_master_initial_offset;         /* Master PSYNC offset. */
    /* Replication script cache. */
    dict *repl_scriptcache_dict;        /* SHA1 all slaves are aware of. */
    list *repl_scriptcache_fifo;        /* First in, first out LRU eviction. */
    unsigned int repl_scriptcache_size; /* Max number of elements. */
    /* Synchronous replication. */
    list *clients_waiting_acks;         /* Clients waiting in WAIT command. */
    int get_ack_from_slaves;            /* If true we send REPLCONF GETACK. */
    ...
```

repl_backlog存储master上的操作命令，只需要将这些命令传给slave。实际上slave本质上它也是一个redisClient。

在命令发送后，最终会调用call函数，如果会产生数据更新，则需要调用propagate(c->cmd,c->db->id,c->argv,c->argc,flags);此函数会判断是否需要aof持久化和主从复制。

在replicationFeedSlaves函数中，将命令存储到repl_backlog中。

```
        for (j = 0; j < argc; j++) {
            long objlen = stringObjectLen(argv[j]);

            /* We need to feed the buffer with the object as a bulk reply
             * not just as a plain string, so create the $..CRLF payload len
             * and add the final CRLF */
            aux[0] = '$';
            len = ll2string(aux+1,sizeof(aux)-1,objlen);
            aux[len+1] = '\r';
            aux[len+2] = '\n';
            feedReplicationBacklog(aux,len+3);
            feedReplicationBacklogWithObject(argv[j]);
            feedReplicationBacklog(aux+len+1,2);
        }
    /* Write the command to every slave. */
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        redisClient *slave = ln->value;

        /* Don't feed slaves that are still waiting for BGSAVE to start */
        if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START) continue;

        /* Feed slaves that are waiting for the initial SYNC (so these commands
         * are queued in the output buffer until the initial SYNC completes),
         * or are already in sync with the master. */

        /* Add the multi bulk length. */
        addReplyMultiBulkLen(slave,argc);

        /* Finally any additional argument that was not stored inside the
         * static buffer if any (from j to argc). */
        for (j = 0; j < argc; j++)
            addReplyBulk(slave,argv[j]);
    }
```


