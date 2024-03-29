# mysql索引结构

## 什么是索引？索引的作用

索引是一种提升查找效率的**一种数据结构**。

在**没有索引时**，查找一条数据的时间往往不可控，消耗的时间会很久，加上数据可能存在磁盘，每多一次IO消耗的时间就更多了。

如果能够按照一定的结构来组织存储数据，那么就可以利用那些数据结构的特性来提高效率。

例如使用hash方式，在没有hash冲突时时间复杂度为O(1)、使用二叉搜索树时间复杂度为O($log_{2}n$)等等，当然，这里说的搜索效率都是内存意义上的。而对于磁盘索引，减少IO次数更为关键，磁盘IO所花费的时间跟内存不是同一个数量级。

## mysql中的索引

mysql中的索引是一种排好序的数据结构。它的实现跟存储引擎有关，不同的存储引擎支持的索引结构不同。

1. 由于需要维护一个结构来提升效率，所以索引会占用额外的空间。

2. 由于需要事先维护这一结构，所以使用索引会减慢增删改的速度。

3. 由于索引事先排好序，在要求分组和排序的场景下可以显著提升效率。

   

