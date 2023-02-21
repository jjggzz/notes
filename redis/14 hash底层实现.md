# hash底层实现

redis的hash结构底层是由**ziplist**和**hash table**两种结构实现的，在数据量较少时使用ziplist作为其实现，当数据量变多时转换为hash table实现，这种转换是单向的，它由以下两个参数控制。

```tex
## hash中元素个数，超过则由ziplist转换为hash table
hash-max-ziplist-entries 512
## hash中元素大小，超过则由ziplist转换为hash table
hash-max-ziplist-value 64
```

前面说过在redis中一切皆redisObject，当type是hash时，它的encoding有两种：OBJ_ENCODING_ZIPLIST或OBJ_ENCODING_HT

```c
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

要了解hash的真正实现就需要了解redis中ziplist的结构和hash table的结构是怎样的

## ziplist结构

如果只使用hash table结构那么在数据量少的时候可能会造成很多的空间浪费，存储的真正数据可能还没有元数据占的空间多。所以在数据量小的时候提供一种**紧凑的结构**，大大减少了空间消耗，这种结构在**内存分配上是在一片连续的空间中**，节约空间的同时减少内存碎片的产生。这是一种典型的空间换时间的做法，具体实现如下（仅仅介绍大致的结构，细节部分略过）

1. 开始的4个字节称为zlbytes，存储整个ziplist所占的字节数（大尾存储，高位在后面）
2. 接下来的4个字节称为zltail，存储的是最后一个条目的内存偏移量（大尾存储，高位在后面）
3. 接下来的两个字节称为zllen，存储整个ziplist中条目个数（大尾存储，高位在后面）
4. 接下来的部分存储一个一个的条目，条目被称为zlentry，条目大小是不确定的，条目的头部会存储上一个条目的大小，当前条目的大小，可以通过这两个值进行内存偏移量的操作，从而在条目间移动
5. 末尾的1个字节称为zlend，存储0XFF，标志着ziplist到达了结尾

```c
 *  [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
 *        |             |          |       |       |     |
 *     zlbytes        zltail     zllen    "2"     "5"  zlend
```

ziplist的总体结构介绍完成了，接下来大致了解一下zlentry的结构：

```c
* 假如前面一个zlentry的长度为2，存储一个Hello World字符串，结构大致如下
* 第一个02代表前一个条目的大小
* 第二个0b代表当前条目的大小
* 向前移动时，在当前条目的开始位置减去2，得到的值就是上一个条目开始的位置
* 向后移动时，在当前条目的开始位置加2（头部的两个值），再加上11（0b），得到下一个条目的开始位置
* [02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]
```

**从这样的结构中可以看出ziplist的优点：**

1. 与传统的链表相比，省去了前后指针所占用的空间，而在C语言中指针大小一般是long型占8个字节
2. 在条目数量较小时（不超过两个字节大小）可以直接获取长度，超过的话需要进行条目遍历
3. 保存了尾部的偏移量，可以直接在尾部进行元素添加与删除操作，这与链表的功能差不多

**但是它也有缺点，毕竟是一种时间换空间的做法，在性能上可能会有点问题：**

1. 由于所有存储的数据都是偏移量，所以要求必须存储在连续的内存空间中
2. 每次进行条目修改或者增加删除时，需要重新分配内存，ziplist在较大时效率会比较低
3. zlentry结构中存储前置条目大小的字段是可变的，如果前置条目小于254则使用1个字节，如果大于254则这个字节固定存储值为254，并且后面会有5个字节来存储更大的数，如果一开始，所有条目的大小是253正好处于临界范围，然后第一个条目变成了大于254，由于这一变化，它的下一个条目需要进行扩容，依此类推，灾难发生了

## hash table结构

redis的hash在不满足hash-max-ziplist-entries**或**hash-max-ziplist-value时，编码格式将会由ziplist转换为hash table。hash table没什么特别的，可以通过java中的HashMap来理解，不过在底层实现上有一点不同，HashMap，在解决冲突上是使用的链表+红黑树，**而hash table在冲突时就只是使用链表**。

```c
// 可以简单的理解为HashMap中的Node
// 这个 *val就是值，它其实是一个redisObject
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

// 可以简单即为HashMap中的Node数组
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

// 可以简单的理解为HashMap
// 这里看到dictht是一个长度为2的数组
// 一般来讲只使用ht[0]，ht[1]在扩容时使用
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

