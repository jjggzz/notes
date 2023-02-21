# list底层实现

redis中的list的实现是基于一种名为**quicklist**的结构，它是一种双向链表结构，每个节点存储的不是一个个的元素，而是一个ziplist（ziplist参考hash底层实现一章），ziplist里面才真正存储元素。

 ```c
 typedef struct quicklist {
   	// 链表的头节点
     quicklistNode *head;
   	// 链表的尾节点
     quicklistNode *tail;
     unsigned long count;        /* total count of all entries in all ziplists */
     unsigned long len;          /* number of quicklistNodes */
     int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
     unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
     unsigned int bookmark_count: QL_BM_BITS;
     quicklistBookmark bookmarks[];
 } quicklist;
 
 // 节点的结构体
 typedef struct quicklistNode {
   	// 前驱节点指针
     struct quicklistNode *prev;
   	// 后继节点指针
     struct quicklistNode *next;
   	// ziplist的指针
     unsigned char *zl;
     unsigned int sz;             /* ziplist size in bytes */
     unsigned int count : 16;     /* count of items in ziplist */
     unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
     unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
     unsigned int recompress : 1; /* was this node previous compressed? */
     unsigned int attempted_compress : 1; /* node can't compress; too small */
     unsigned int extra : 10; /* more bits to steal for future usage */
 } quicklistNode;
 ```

list受两个参数控制，其中一个控制何时创建节点，另外一个控制quicklist的压缩

```tex
## 控制节点中ziplist的大小
## 如果设置为正数，那么指的是ziplist中条目的个数，如果超过，则新建一个quicklistNode
## 对于负数则是指定最大大小，如果超过，则新建一个quicklistNode
## 作者推荐-2或者-1是比较好的
## -5: max size: 64 Kb  <-- not recommended for normal workloads
## -4: max size: 32 Kb  <-- not recommended
## -3: max size: 16 Kb  <-- probably not recommended
## -2: max size: 8 Kb   <-- good
## -1: max size: 4 Kb   <-- good
list-max-ziplist-size -2
## 列表压缩深度，默认为0的话代表不对列表进行压缩
## 值为1代表左右头节点不压缩，其余内容压缩
## 值为2代表左右两个节点不压缩，其余内容压缩
## 依此类推
## 可以看到，无论配置什么值头尾节点都是不压缩的
list-compress-depth 0
```

