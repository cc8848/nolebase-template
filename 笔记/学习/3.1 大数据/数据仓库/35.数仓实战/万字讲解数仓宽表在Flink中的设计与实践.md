- [万字讲解数仓宽表在Flink中的设计与实践 (qq.com)](https://mp.weixin.qq.com/s/1-EdG2j5R1dbU1lIXKW4sg)

## 1 宽表基础知识

### 1.1 基本概念

数据仓库模型一般有四种：`宽表模型`，`星型模型`，`雪花模型`，`星座模型`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKibRDCicAtiaxxfmweuAwKVhgdfRQz4qGm6mBGyPZ6SlND9jJUzQiaYX9VmQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

四种模型定义如下：

**1）宽表模型（单例模型）**

通常是指业务主体相关的`指标、维度、属性`关联在一起的一张数据库表。

**2）星型模型**

由`事实表和维度表`组成，一个星型模型中可以有`一个或者多个`事实表，每个事实表引用任意数量的`维度表`。星型模式的物理模型像一颗星星的形状，中心是一个事实表，围绕在事实表周围的维度表表示星星的放射状分支，这就是星型模式这个名字的由来。

**3）雪花模型**

雪花模式也是由`事实表`和`维度表`所组成，它将低基数的属性从维度表中移除并形成单独的表。基数指的是一个字段中不同值的个数，如性别这样的列基数就很低。在雪花模式中，一个维度被规范化成多个关联的表，而在星型模式中，每个维度由一个`单一的`维度表所表示。星型模式是雪花模式的一个特例(维度没有多个层级)。

**4）星座模型**

由`多个事实表`组合，维表是公共的，可以被多个事实表共享。

### 1.2 宽表优缺点

在数据仓库建设中，组织相关和相似数据，采用`明细宽表`，减少数据扫描，提高明细数据表的易用性，以及查询性能。

然而宽表也存在一些缺点：

- 数据冗余，由于把不同的内容都放在同一张表存储，宽表已经不符合`三范式`的模型设计规范，随之带来的主要坏处就是数据的`大量冗余`，对存储是个挑战。
- 灵活性差，比如线上业务表结构`变更`，宽表模式`改造量`也比较大。
- 开发流程变长，我们需要去了解`业务全流程`，得需要知道需扩展哪些维度，沉淀哪些指标，这样就流程会比较长，特别是有些业务快速迭代的话，就有点捉襟见肘。

### 1.3 怎么判断是宽表好还是多维表好？

数据仓库每张表的搭建，主要依赖于这个表在整个数据仓库中的`作用和相关意义`。首先要清楚这个表的存在是为了`解决那些问题`，`什么角色使用`，`怎么保证使用者尽可能好的体验`解决问题。从以上所提到的角度去看待问题，拆解以下几点因素：

- 1）多表关联查询的使用频次有多高，将重复高频的事情简化，是不是更好？
- 2）查询体验上需要考虑多表关联之后的查询性能问题，关联表，是否影响查询速度？
- 3）某些场景下，是否存在不能打宽的情况，比如事实维度一对多场景

### 1.4 策略总结

从规范化的角度来讲，数据仓库的设计者是希望`越规范越好`，因为这样会减少数据的冗余，而且也便于模型的扩展。

从反规范化的角度来讲，数据仓库的使用者是希望使用`越方便越好`，他们并不太关系规范不规范冗余不冗余，只要用着方便就好。

这种情况在工作中是十分常见的，那么该怎样来解决它？下面有两个思路：

**1）两种方式都存**

虽然，这样看起来会占用更多的存储空间，但不失为一种合适的解决方案，因为宽表是通过别的表拼接而成的，因此宽表的`存储周期是可以短一些`。

**2）只存多个维度表**

通过`视图`来创建宽表。这种方式适合于宽表的`查询次数较少`的情况。比如在Hive中，宽表其实只是为了计算出来之后导入Es等系统中供其它系统查询，那么久没必要存储一份宽表，直接通过视图来封装就可以。

鉴于以上的一些考量，在数仓底层可以使用`星型模型`，而宽表则可以存在于更靠后的层次。

## 2 如何设计宽表

宽表的好处和不足我们都讲了，也就是说宽表虽好，但是带来的问题也很多，下面我们就看一下如何从设计的角度来避免宽表的不足之处。

### 2.1 宽表到底多宽

我们要思考到底多宽才合适的，我们可以反着思考，宽表`不应该包含什么数据`，这个问题很好回答，宽表不应该包含不属于它所在域的数据，例如会员域的宽表只应该包含会员相关的信息，同理我们的宽表是针对某一个域而言的，也就是说它是`有边界的`。

所以可以大胆点说宽表只要`不跨域`，并且`方便使用`都是合理的。但是这似乎并不能解决我们上面提到的宽表的不足，只是指明了宽表的一个大致的方向。有了方向之后我们通过我们的设计策略就可以让宽表瘦下来。

### 2.2 宽表设计策略

设计策略主要包括：主次分类、冷热分离、稳定与不稳定分离。

**1）主次分类**

假设我们做的是一个会员域的宽表，但是会员域是还是一个比较大的概念，所以我们还要发掘出我们这个表的主题。

例如我们做的是一张`会员域`下的`会员基本信息宽表`，那么我们专注的肯定就是基本信息，例如`会员信息打通`。当然因为宽表你可能还会冗余的其他信息进来，但是当这样的信息越来越多的时候，我们这张表的主题就越来越弱，所以我们就需要`做拆分`。

拆分可以让我们更加聚焦表的主题，对于数仓开发人员而言可以更好的维护、对于使用方而言可以更加清楚的理解这张表的主题。

**2）冷热分离**

假设我有一张宽表，里面有`200个`字段，有`30张报表`在使用它，但是我发现前面150个经常字段经常被使用，后面 50个字段只有一两张报表使用到了，那么我们就可以做一个冷热分离，将`宽表拆分`。

**3）稳定与不稳定分离**

我们经常有这样的宽表，它依赖埋点数据，但是我们的埋点数据的特点就是量大，导致计算`经常延迟`，那么我们的宽表就会受影响，从而我们的报表就受影响，但是很多时候你发现报表根本没有用过埋点计算出来的指标，或者是只用了一两个。

那我们可以将其拆分，如果报表没有使用到那就最好了，如果使用到了，那就后推，在报表层面上做关联，这样我们的埋点数据即使出不来，我们的报表数据还是可以看的。

## 3 Flink中的宽表实现

下面内容均来自：阿里基于 Flink SQL 构建流批一体的 ETL 数据集成。

数据打宽是数据集成中最为常见的业务加工场景，数据打宽最主要的手段就是`Join`，Flink SQL 提供了丰富的 Join 支持，包括` Regular Join`、`Interval Join`、`Temporal Join`。

这些JOIN技术细节老兵也单独讲过一期文章，详情查看：

[面试高频 | 你追我赶的Flink双流join](http://mp.weixin.qq.com/s?__biz=MzI3NTU5ODA2NA==&mid=2247498708&idx=1&sn=637aedf1f046947058fdffd1f9c5d262&chksm=eb00e44bdc776d5d244db7d033f6e84771d74dab445bfa13b3441bc9a5911beafa4e209af67b&scene=21#wechat_redirect)

### 3.1 Regular join

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKib98ZOiaDYHMJQt27rtBicUJgnmib25ECiasM9llQUiaUFGO6gf1sRjsKYc7A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> Regular Join 就是大家熟知的双流 Join，语法上就是普通的 JOIN 语法。图中案例是通过广告曝光流关联广告点击流将广告数据打宽，打宽后可以进一步计算广告费用。

- 从图中可以看出，曝光流和点击流都会存入 join 节点的 state，join 算子通过关联曝光流和点击流的 state 实现数据打宽。
- Regular Join 的特点是，任意一侧流都会触发结果的更新，比如案例中的曝光流和点击流。同时 Regular Join 的语法与传统批 SQL 一致，用户学习门槛低。
- 但需要注意的是，Regular join 通过 state 来存储双流已经到达的数据，state 默认永久保留，所以 Regular join 的一个问题是默认情况下 state 会持续增长，一般我们会结合 state TTL 使用。

### 3.2 Interval Join

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKibWjK7vOsL3QnhDkkSYb9N0ErW8AaoXMen5tFKhrU6IKic78fpIUzL5tQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> Interval Join 是一条流上需要有时间区间的 join，比如刚刚的广告计费案例中，它有一个非常典型的业务特点在里面，就是点击一般发生在曝光之后的 10 分钟内。

- 相对于 Regular Join，我们其实只需要关联这10分钟内的曝光数据，所以 state 不用存储全量的曝光数据，它是在 Regular Join 之上的一种优化。
- 要转成一个 Interval Join，需要在两个流上都定义时间属性字段（如图中的 click_time 和 show_time）。并在 join 条件中定义左右流的时间区间，比如这里我们增加了一个条件：点击时间需要大于等于曝光时间，同时小于等于曝光后 10 分钟。
- 与 Regular Join 相同， Interval Join 任意一条流都会触发结果更新，但相比 Regular Join，Interval Join 最大的优点是 state 可以自动清理，根据时间区间保留数据，state 占用大幅减少。
- Interval Join 适用于业务有明确的时间区间，比如曝光流关联点击流，点击流关联下单流，下单流关联成交流。

### 3.3 Temporal Join

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKib9B2kbYicLIVVDlx1r9v1QFCSiaTniaZ7libj3ExgJvGBV4fmiaa5Tquoia9w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> Temporal join (时态表关联) 是最常用的数据打宽方式，它常用来做我们熟知的维表 Join。在语法上，它需要一个显式的 FOR SYSTEM_TIME AS OF 语句。

- 它与 Regular Join 以及 Interval Join 最大的区别就是，维度数据的变化不会触发结果更新，所以主流关联上的维度数据不会再改变。
- Flink 支持非常丰富的 Temporal join 功能，包括关联 lookup DB，关联 changelog，关联 Hive 表。在以前，大家熟知的维表 join 一般都是关联一个可以查询的数据库，因为维度数据在数据库里面，但实际上维度数据可能有多种物理形态，比如 binlog 形式，或者定期同步到 Hive 中变成了 Hive 分区表的形式。在 Flink 1.12 中，现在已经支持关联这两种新的维表形态。

**1）Temporal Join Lookup DB**

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKibwRibKA64m2KAOjBfEmSsOtJmjmkIEEyNtcIfWSvcZH6XT5xwBpy3tBg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> Temporal Join Lookup DB 是最常见的维表 Join 方式，比如在用户点击流关联用户画像的案例中，用户点击流在 Kafka 中，用户实时画像存放在 HBase 数据库中，每个点击事件通过查询并关联 HBase 中的用户实时画像完成数据打宽。

Temporal Join Lookup DB 的特点是，维表的更新不会触发结果的更新，维度数据存放在数据库中，适用于实时性要求较高的场景，使用时我们一般会开启 Async IO 和内存 cache 提升查询效率。

**3）Temporal Join Binlog**

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKib8GOLfEP2gntNKJriaANdjmyVdjVUNoXpPtmY8NWFZVhtngicWAHRXS2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> 通过从 changelog在 Flink state 中物化出维表来实现维表关联。，我们可以通过 Flink CDC connector 把直播间数据库表的 changelog 同步到 Kafka 中。

- 右边这段 SQL我们用了 upsert-kafka connector 来将 MySQL binlog 写入了 Kafka，也就是 Kafka 中存放了直播间变更数据的 upsert 流。然后我们将互动数据 temporal join 这个直播间 upsert 流，便实现了直播数据打宽的功能。
- 这里 FOR SYSTEM_TIME AS OF 不是跟一个 processing time，而是左流的 event time，它的含义是去关联这个 event time 时刻的直播间数据，同时我们在直播间 upsert 流上也定义了 watermark，所以 temporal join changelog 在执行上会做 watermark 等待和对齐，保证关联上精确版本的结果，从而解决先前方案中关联不上的问题。

**3）Temporal Join Hive**

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKibIromnXCadaic8vOibsdT0GVricf6aCrxMb0jFauBvGcYrb7r4nUaaRCCg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> 在数仓场景中，Hive 的使用是非常广泛的，Flink 与 Hive 的集成非常友好，现在已经支持 Temporal Join Hive 分区表和非分区表。

我们举个典型的关联 Hive 分区表的案例：订单流关联店铺数据。店铺数据一般是变化比较缓慢的，所以业务方一般会按天全量同步店铺表到 Hive 分区中，每天会产生一个新分区，每个分区是当天全量的店铺数据。

- 为了关联这种 Hive 数据，只需我们在创建 Hive 分区表时指定右侧这两个红圈中的参数，便能实现自动关联 Hive 最新分区功能，partition.include = latestb 表示只读取 Hive 最新分区，partition-name 表示选择最新分区时按分区名的字母序排序。
- 到 10 月 3 号的时候，Hive 中已经产生了 10 月 2 号的新分区, Flink 监控到新分区后，就会重新加载10月2号的数据到 cache 中并替换掉10月1号的数据作为最新的维表。
- 之后的订单流数据关联上的都是 cache 10 月 2 号分区的数据。Temporal join Hive 的特点是可以自动关联 Hive 最新分区，适用于维表缓慢更新，高吞吐的业务场景。

### 3.4 总结

![图片](https://mmbiz.qpic.cn/mmbiz_png/lvcHBkdwcibyPAoiaDlUNgKcqI7JticYicKibBsj4VAc3levOPib7pXystiazM5en5ncziaIdKE4UN7Ma61zpibiabG2QOwA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

总结一下我们刚刚介绍的几种在数据打宽中使用的 join：

- Regular Join 的实效性非常高，吞吐一般，因为 state 会保留所有到达的数据，适用于双流关联场景；
- Interval Jon 的时效性非常好，吞吐较好，因为 state 只保留时间区间内的数据，适用于有业务时间区间的双流关联场景；
- Temporal Join Lookup DB 的时效性比较好，吞吐较差，因为每条数据都需要查询外部系统，会有 IO 开销，适用于维表在数据库中的场景；
- Temporal Join Changelog 的时效性很好，吞吐也比较好，因为它没有 IO 开销，适用于需要维表等待，或者关联准确版本的场景；
- Temporal Join Hive 的时效性一般，但吞吐非常好，因为维表的数据存放在cache 中，适用于维表缓慢更新的场景，高吞吐的场景。