# redis数据结构基本架构

redis在设计上将所有的数据都抽象成了dictEntry，在dictEntry中，key的类型是内嵌字符串（sds），value被抽象成了redisObject，而那五大数据结构可以简单的理解为redisObject派生出来的子类型

```c
// dictEntry结构
typedef struct dictEntry {
    void *key; // 数据key
    union {
        void *val; // redisObject指针
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */

// redisObject结构
typedef struct redisObject {
    unsigned type:4; // 数据的类型string、list、hash、set、zset
    unsigned encoding:4; // 数据在底层的编码格式
    unsigned lru:LRU_BITS; // 最近的访问时间，用做淘汰策略
    int refcount; //引用计数
    void *ptr; // 指向真正的数据结构，sds、双端链表、压缩列表、跳表、hash等等具体的数据结构
} robj;
```
