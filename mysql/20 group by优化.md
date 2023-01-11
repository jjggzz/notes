# group by优化

1. group by的索引使用规则几乎和order by一致，group by即使过滤条件没有用到索引，group by也可以用上索引。

2. group by是先排序再分组

3. where的性能高于having

4. 如果需要使用order by、group by、distinct这些语句，请将结果集控制在1000以内，否则sql会非常慢

   