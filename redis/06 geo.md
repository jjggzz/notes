# geo

对于有位置信息查询或者范围搜索的需求，如果仅仅使用mysql是没法实现的，比如说我想搜索以当前位置为圆形，半径10公里内的所有超市，对于这种需求sql无能为力，即使使用范围查询，搜索结果也是一个矩形的，并且他只是平面直角坐标的二位平面。

所以需要有支持地理位置功能的组件来提供这些支持，es支持geo相关的搜索与查询，现在redis也支持这些功能了。

geo底层其实是zset，它并没有引入新的数据结构，而是利用geohash将经纬度编码成0101的二进制串，这样位置靠近的两个点就会具有相同的前缀，方便进行前缀查询。而zset的天然排序会使得具有相同前缀的值靠在一起，范围查询也方便

## geo的使用

```shell
## 向city:guandong中添加条数据
## sz的经纬度为114.068002 22.548456
## gz的经纬度为113.270279 23.125765
6379> geoadd city:guandong 114.068002 22.548456 sz 113.270279 23.125765 gz
## 返回sz的经纬度
6379> geopos city:guandong sz
## 返回sz到gz的距离，单位km
## 单位可选 m/lm/ft/mi
6379> geodist city:guandong sz gz km
## 查询city:guandong中以坐标114,22为圆点，半径100km内的点
## 可选参数：
## WITHCOORD：输出找到的点的经纬度
## WITHDIST：输出到圆点的距离
## COUNT：指定返回的条数
## ASC|DESC：指定排序顺序
## STORE：输出到指定的key
6379> georadius city:guandong 114 22 100 km
## 查询city:guandong中以zs坐标为圆点，半径100km内的点
## 可选参数与georadius命令相同
6379> georadius city:guandong sz 100 km
```

