# bitmap

## bitmap介绍

bitmap直译位图，它底层其实是string，只不过是将字符串的每个字符都拆成八位使用（ascii码，一个字符占一个字节，八位），这些字节数组扩容的方式是8的倍数进行扩容的，它支持2^32次方个二进制位的设置。

## bitmap的使用

```shell
## 设置位数组的第0个位置的值为1
6379> setbit key1 0 1
## 获取位数组第0个位置的值
6379> getbit key1 0
## 统计到底有多少个位被设置成1
6379> bitcount key1
```

```shell
## 设置位数组的值01100001 => 97
6379> setbit key1 1 1
6379> setbit key1 2 1
6379> setbit key1 7 1
## 通过字符串的获取方式获取，打印出来的是a
## 这也印证了底层其实是string
6379> get key1
```

```shell
## 设置位数组第2个位置的值为1
6379> setbit key1 1 1
## 设置位数组第3个位置的值为1
6379> setbit key1 2 1
## 设置位数组第11个位置的值为1
6379> setbit key1 10 1
## 统计位数组有多少个位被设置为1 => 3
6379> bitcount key1
## 使用统计字符串长度的方式获取这个key的长度，发现长度为2（在设置第11个位置时，原先的大小为8的位驻数组载不下，只能扩容，每次加8，所以是16个位，最终计算出来是两个字节）
6379> strlen key1
```

```shell
## 对key1 key2两个key做与运算，得到的结果放到desckey中
## key1 0000 0001
## key2 1000 0001
## desckey 0000 0001
6379> bitop and desckey key1 key2
## 对key1 key2两个key做或运算，得到的结果放到desckey2中
## key1 0000 0001
## key2 1000 0001
## desckey 1000 0001
6379> bitop or desckey2 key1 key2
## 对key1做非运算，得到的结果放到desckey3中
## key1 0000 0001
## desckey 1111 1110
6379> bitop not desckey3 key1
## 对key1 key2两个key做异或运算，得到的结果放到desckey4中
## key1 0000 0001
## key2 1000 0001
## desckey 1000 0000
6379> bitop xor desckey4 key1 key2
```

