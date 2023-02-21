# zset的底层实现

redis的zset编码格式有两种**ziplist**和**skiplist+hash table**，在元素较少的时候使用ziplist这种压缩结构以节省内存，在元素较多的时候使用skiplist+hash table实现，ziplist不用多说在hash table一章介绍过。

```c
## zset中元素超过128则变换结构
zset-max-ziplist-entries 128
## 每个元素大于64字节则变换结构
zset-max-ziplist-value 64
```

对于skiplist+hash table的实现结构，hash table用做value到分数的映射，使得获取值和分数在O(1)的时间范围内。

skiplist主要解决按分数查询的效率问题，skiplist是一种查询时间在O(logn)的数据结构。一想到O(logn)就会想到二分法，二叉搜索树，红黑树，多路搜索树等等，但是为什么redis不选择这些数据结构而是选择跳表呢？

1. 排好序的顺序表，使用二分法可以达到O(logn)的查询效率，但是如果数据量比较大，难以找到那么大块的连续内存
2. 二叉搜索树，如果数据插入是有序的那么就会出现偏斜，导致查询效率退化到O(n)
3. 红黑树，在数据插入到时候需要进行旋转，变色等操作，并且如果有范围搜索的话不好弄，实现起来也难
4. 多路搜索树，同样存在叶分裂等操作（参考mysql的B+tree，mysql使用B+tree是因为需要进行磁盘IO，B+tree的高度比跳表低，跳表被秒杀），而在内存中不存在磁盘IO的问题，高一点也没事

**虽然采用跳表存储可能会多花费一些空间，但是综合实现难易程度、区间查找性能，选择了跳表**

## skiplist介绍

skiplist是一种基于链表，并且要求链表中内容有序的数据结构，通过将内容抽取升层的方式构建。除了最底层外，每一层的元素是下一层的二分之一，在元素插入时，首先会插入最底层，然后有50%的概率升到第二层，同时有25%的概率升到第三层，依此类推直到2^-63^

```tex
				1-------------------------------------------25
				1------------------>10----------------------25
			  1-------->6-------->10--------->19--------->25
head--->1--->3--->6--->8--->10--->15--->19--->22--->25--->tail
```

如果我要考察22这个节点，只需要比较5次，可以看到效率是得到很大的提升的，如果元素再多一点提升会更明显

## redis的skiplist

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    // 元素
  	sds ele;
  	// 分数
    double score;
  	// 后退指针，反向操作时使用
    struct zskiplistNode *backward;
    struct zskiplistLevel {
      	// 前进指针
        struct zskiplistNode *forward;
      	// 跨度
        unsigned long span;
    } level[];
  	
} zskiplistNode;

typedef struct zskiplist {
  	// 头尾指针
    struct zskiplistNode *header, *tail;
  	// 链表长度
    unsigned long length;
  	// 当前最高层数是几
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

