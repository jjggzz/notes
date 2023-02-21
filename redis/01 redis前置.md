# redis前置

## 如何查看redis版本

方法一：

```shell
## 直接在shell环境使用该命令
redis-server -v
```

方法二：

```shell
## redis-cli登陆到redis中
redis-cli
## 使用info server获取redis的版本号
info server
```

## 如何让redis-cli显示中文

```shell
## 加上--raw参数
redis-cli --raw
```

## 为什么redis集群只有16384个槽？

按道理redis使用crc16校验算法计算key应该放置的槽位，crc16结果取值范围应该在0～65535之间，为什么不是65535呢？

由于redis集群间的需要进行心跳通信，而心跳包中占空间最大的就是当前节点负责的slot信息，他是用bitmap来存储的，负责哪些slot对应的位就会被标记，如果是65535，那么需要65535 / 8 / 1024 = 8K的存储空间，如果是槽位限制在16384，那么只需要16384 / 8 / 1024 = 2K的空间，数据包大小相差4倍。

再有一个原因就是redis集群节点越多通信成本越高，redis作者不建议redis集群节点超过1000，因为这会造成网络拥堵。对于1000以内数量的集群，16384个槽够用了

