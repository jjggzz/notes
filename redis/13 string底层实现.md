# string底层实现

string类型在编码格式（所谓的编码格式就是底层的数据结构）上有三种，分别是**int**、**embstr（sds）**、**row（sds）**。

1. 当存储的是整数并且值处于long类型可以表示的数值范围内时编码格式为int
2. 其他情况，并且长度小于44字节时编码格式为embstr
3. 剩下的情况编码格式使用row格式

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
    unsigned type:4; // 数据的类型 string
    unsigned encoding:4; // 数据在底层的编码格式 1、8、0
    unsigned lru:LRU_BITS; // 最近的访问时间，用做淘汰策略
    int refcount; //引用计数
    void *ptr; // 指向真正的数据结构
} robj;
```

## sds是个什么东西？

sds是（Simple dynamic string）的简称，从注释上可以看出它是一个内嵌的字符串类型，redis的作者在实现redis时使用到字符串并没有直接使用C语言本身的字符串实现（字符数组）来作为字符串的载体，而是自己实现了一套内嵌的字符串实现，这一套是构建在字符数组之上的。

```c
// 拿一个简单的结构来说
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */ // 使用的长度
    uint8_t alloc; /* excluding the header and null terminator */ // 分配的长度，不包括头和结束符
    unsigned char flags; /* 3 lsb of type, 5 unused bits */ // 标识类型，标志到底是sds8还是sds16等等
    char buf[]; //真正存内容的地方
};

```

redis中有sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64等sds类型，其中sdshdr5未被使用，后缀数字分别代表最多可存储2的几次方（单位是字节）长度的字符串

## 为什么要弄一个sds出来呢？

要理解为什么要引入sds，可以从多的几个参数来理解。

1. len表示使用的长度，这样在统计字符串长度的时候时间复杂度就在O(1)，没有这个值的话需要遍历字符数组，时间复杂度在O(n)
2. alloc表示已分配的长度，那么就可以通过它来计算还有多少空间是未使用的（free），从而引入预分配算法
   1. redis的字符串长度是变化的，如果设置一个固定的长度，那么很可能出现数组越界
   2. 在字符串变长时可以根据这个值判断是否要进行内存申请，避免数据越界，并且可以预先分配一些空间，减少空间的分配次数
   3. 那么记录已分配的大小就很重要了，可以计算出free的值，在字符串缩短的时候，只需要修改长度即可，当长度再次变长时，可以复用之前分配的空间，减少内存分配次数，假如再提供相应的函数进行内存回收，就不会造成内存泄漏

3. 通过flags可以知道是什么类型，从而知道最大的内存分配上限是多少也可以用它来做内存计算
4. 他是二进制安全的，引入len之后，读取数据时直接读取需要的长度，如果没有这个，在读取char数组内容时遇到结束符会停下

## string底层的一些细节

redis在使用string存储整型数据时，如果值在0-10000以内则使用共享变量（类似于java的Integer），此时所有值在这个范围内的*ptr都指向这个共享变量，需要注意的是对于整数 *ptr直接存的是值，而不是值所存在的地址

如果不能转换为整形值，但是长度小于44则使用embstr来存储。为什么是44？应该跟内存分配器有关，它最大分配64字节，如果是这样的话 64 - 16 - 3 = 45，所以buf[]长度最好是45，去掉一个末尾的结束符正好是44，这样它和redisObject就可以放在一个连续的空间中了

```c
// 4bit + 4bit + 24bit + 4字节 + 8字节 = 16字节
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;

// 1字节 + 1字节 + 1字节 = 3字节
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

如果对embstr做修改（embstr是只读的，只要修改都会变成raw，无论长度）或者长度大于44，会使用raw编码格式，这种格式不会与redisObject放在连续的空间中，并且在分配时会做一个判断，如果已分配但是未使用的空间 > 字符串长度的十分之一，则进行回收，减少内存浪费

