# set底层实现

redis的set是基于**intset**和**hash table**两种结构来实现，的其中hash table在hash底层实现的时候就已经看过，而intset则是一种新的数据结构，它是一种紧凑的数据结构，当**其中元素都为整数并且在long类型范围内，并且元素个数小于set-max-intset-entries配置**的值时采用，当不满足时会转换为hash table，转换是单向的。

```tex
## 当元素超过512个时，转换为hash table
set-max-intset-entries 512
```

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
  	// 编码格式默认情况下是INTSET_ENC_INT16
    uint32_t encoding;
  	// 元素个数
    uint32_t length;
  	// 元素内容，虽然声明是int8_t，但是真正的类型由encoding决定，它是一个数组
    int8_t contents[];
} intset;
```

intset新增元素的时候会判断新增的元素大小超没超过当前encoding可以表示的大小，如果超过则需要将

contents进行升级，具体操作是先申请空间，然后将每个元素都转换成对应的类型，并拷贝。最后在插入新值。

值得一说的是intset是有序的，所以它的查找时间是O(logn)，插入时间是O(n)