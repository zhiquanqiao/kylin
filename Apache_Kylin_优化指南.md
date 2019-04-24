# Apache Kylin 优化指南

[TOC]



## 一. 维度优化

### 为什么需要维度优化

如果不进行任何维度优化，直接将所有的维度放在一个聚集组里，Kylin将会计算所有的维度组合（cuboid）。比如，有12个维度，Kylin 就计算2的12次方即4096个cuboid，实际查询可能用不到cuboid 不到1000个，甚至更少。如果进行围堵优化，会造成集群计算和存储资源的浪费，也会影响cube的build和查询性能。

**当你保存的cube时遇到下面的异常信息时，意味着一个聚合组已经大于4096，你就必须进行维度优化了。**

![image_1aufkne841pr4155d15bh1cmigf09.png-43.3kB](http://static.zybuluo.com/kangkaisen/0rsmuprx31dn0ry1llrda2cr/image_1aufkne841pr4155d15bh1cmigf09.png)

### 如何计算1个聚合组的维度组合数

```
当有层次维度时，公式如下：
(hierarchy.size()+1) * hierarchyDimsList.size() * (1 << jointDimsList.size()) * (1 << normalDims.size())
当没有层次维度时，公式如下：
(1 << jointDimsList.size()) * (1 << normalDims.size())
```

### 如何进行维度优化

确认你设置的cube维度都是你查询时用到的。

维度优化手段

- 聚合组
- 衍生维度
- 强制维度
- 层次维度
- 联合维度
- Extended Column

#### 聚合组

聚合组：用来控制那些cuboid 需要计算

适用场景： 不是只需要计算base cuboid 的情况下，都需要聚合组。

注意事项：一个维度可以出现在多个聚合组中，但是build 期间只会计算一次。

#### 衍生维度

衍生维度：维度中可以由主键导出的维度作为衍生维度。

使用场景：以星型模型接入时。例如用户维度表可以从userid 推导出用户的姓名，年龄，性别。

优化效果：维度表的N个维度组合成的cuboid个数会从2的N次方为2

![此处输入图片的描述](http://static.zybuluo.com/kangkaisen/g9130yn0dvaz9kn4nfneyh2r/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-03-31%20%E4%B8%8B%E5%8D%882.24.07.png)

#### Mandatory维度

强制维度： 所有cuboid必须包含的维度，不会计算不包含cuboi。

适用场景：可以将确定在查询时一定会使用的维度设置为强制维度，例如，时间维度

优化效果：将一个维度设置为强制维度，则cuboid 个数系那个直接减半。

![强制维度](http://static.zybuluo.com/kangkaisen/y744npgiv7ccy36tq6dsfyct/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-03-31%20%E4%B8%8B%E5%8D%882.20.20.png)

#### Hierachy维度

层次维度：具有一定的层次关系的维度

使用场景：像 年，月，日；国家，省份，城市 这类具有层次关系的维度。

优化效果： 将N个维度设置为层次维度，则这N个维度组合成的cuboid 个数会从2的N次方减少到N+1。

![此处输入图片的描述](http://static.zybuluo.com/kangkaisen/984xt2g15whaejhvg9ih3vx6/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-03-31%20%E4%B8%8B%E5%8D%882.21.49.png)

#### Joint维度

联合维度：将几个维度视为一个维度

使用场景： 1 可以将确定在查询时一定会同时使用的几个维度设为一个联合维度。

​				   2 可以将基数很小的几个维度设为一个联合维度。

​                   3 可以将查询时很少使用的几个维度设为一个联合维度。

优化效果：将N个维度设置为联合维度，则这N个维度组合成的cuboid个数会从2的N次方减少到1。

#### Extended Column

在OLAP分析场景中，经常对某个id 进行过滤，但查询结果要展示为name的情况，比如user_id 和 user_name。

这类问题的三种解决方式：

a. 将ID和name 都设置为维度，查询语句类似`select name, count(*) from table where id = 1 group by id,name`。这种方式的问题是会导致维度增多，导致预计算结果膨胀；

b. 将id 和name 都设置为维度，并且将两者联合。这种方式的好处是保持维度组合不会增加，但是限制了维度的其他优化，比如id 不能在设置为强制维度活着是层次维度；

c. 将id 设置为维度，Name设置为特殊的Measure,类型为Extended Column。这种方式既能保证过滤id且查询name的需求。

所以此类需求我们推荐使用 Extended Column。