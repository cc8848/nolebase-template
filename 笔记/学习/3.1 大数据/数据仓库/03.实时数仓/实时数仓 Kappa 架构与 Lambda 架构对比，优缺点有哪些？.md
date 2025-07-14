- [实时数仓 Kappa 架构与 Lambda 架构对比，优缺点有哪些？_一个写湿的程序猿的博客-CSDN博客_kappa架构的优缺点](https://qinjl.blog.csdn.net/article/details/123552852)

# 1 数据仓库概念

数据仓库是一个面向主题的（`Subject Oriented`）、集成的（`Integrate`）、相对稳定的（`Non-Volatile`）、反映历史变化（`Time Variant`）的[数据集](https://so.csdn.net/so/search?q=数据集&spm=1001.2101.3001.7020)合，用于支持管理决策。

数据仓库概念是 Inmon 于 1990 年提出并给出了完整的建设方法。随着互联网时代来临，数据量暴增，开始使用 大数据工具 来替代经典数仓中的传统工具。此时仅仅是工具的取代，架构上并没有根本的区别，可以把这个架构叫做离线大数据架构

后来随着业务实时性要求的不断提高，人们开始在 离线大数据架构 基础上加了一个加速层，使用流处理技术直接完成那些实时性要求较高的指标计算，这便是 `Lambda` 架构。

再后来，实时的业务越来越多，事件化的数据源也越来越多，实时处理从次要部分变成了主要部分，架构也做了相应调整，出现了以实时事件处理为核心的 Kappa 架构。

![在这里插入图片描述](https://img-blog.csdnimg.cn/eec1c06b57704d52985eed789a4aa7fd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_20,color_FFFFFF,t_70,g_se,x_16)

# 2 离线大数据架构

数据源通过离线的方式导入到离线数仓中。下游应用根据业务需求选择直接读取 DM 或加一层数据服务，比如 MySQL 或 Redis。数据仓库从模型层面分为三层：

- ODS，操作数据层，保存原始数据；
- DWD，数据仓库明细层，根据主题定义好事实与维度表，保存最细粒度的事实数据；
- DM，数据集市/轻度汇总层，在 DWD 层的基础之上根据不同的业务需求做轻度汇总；

如果要细分，分为五层：

ODS

- ODS 层：Operation Data Store，数据准备区，贴源层。直接接入源数据的：业务库、埋点日志、消息队列等。ODS 层数数据仓库的准备区

DW

- DWD 层：Data Warehouse Details,数据明细层，属于业务层和数据仓库层的隔离层，把持和 ODS 层相同颗粒度。进行数据清洗和规范化操作，去空值/脏数据、离群值等。
- DWM 层：Data Warehouse middle,数据中间层，在 DWD 的基础上进行轻微的聚合操作，算出相应的统计指标
- DWS 层：Data warehouse service,数据服务层，在 DWM 的基础上，整合汇总一个主题的数据服务层。汇总结果一般为`宽表`，用于 OLAP、数据分发等。

ADS

- ADS 层：Application data service, 数据应用层，存放在 ES，Redis、PostgreSql 等系统中，供数据分析和挖掘使用。

典型的数仓存储是 HDFS/Hive，ETL 可以是 MapReduce 脚本或 HiveSQL。

![在这里插入图片描述](https://img-blog.csdnimg.cn/403a6c529b074ec588e45efdeaeb31ac.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_20,color_FFFFFF,t_70,g_se,x_16)

数仓分层的优点：

1. 划清层次结构：每一个数据分层都有它的作用域，这样我们在使用表的时候能更方便地定位和理解。
2. 数据血缘追踪：简单来讲可以这样理解，我们最终给下游是直接能使用的业务表，但是它的来源有很多，如果有一张来源表出问题了，我们希望能够快速准确地定位到问题，并清楚它的危害范围。
3. 减少重复开发：规范数据分层，开发一些通用的中间层数据，能够减少极大的重复计算。
4. 把复杂问题简单化：将一个复杂的任务分解成多个步骤来完成，每一层只处理单一的步骤，比较简单和容易理解。而且便于维护数据的准确性，当数据出现问题之后，可以不用修复所有的数据，只需要从有问题的步骤开始修复。
5. 屏蔽原始数据的异常：屏蔽业务的影响，不必改一次业务就需要重新接入数据。

# 3 Lambda 架构

随着大数据应用的发展，人们逐渐对系统的实时性提出了要求，为了计算一些实时指标，就在原来离线数仓的基础上增加了一个实时计算的链路，并对数据源做流式改造（即把数据发送到消息队列），实时计算去订阅消息队列，直接完成指标增量的计算，推送到下游的数据服务中去，由数据服务层完成离线&实时结果的合并。

Lambda 架构（Lambda Architecture）是由 Twitter 工程师南森·马茨（Nathan Marz）提出的大数据处理架构。这一架构的提出基于马茨在 BackType 和 Twitter 上的分布式数据处理系统的经验。

Lambda 架构使开发人员能够构建大规模分布式数据处理系统。它具有很好的灵活性和可扩展性，也对硬件故障和人为失误有很好的容错性。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5e6252582b324f54b3970b5f954de793.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_14,color_FFFFFF,t_70,g_se,x_16)

Lambda 架构总共由三层系统组成：`批处理层（Batch Layer）`，`速度处理层（Speed Layer）`，以及用于响应查询的`服务层（Serving Layer）`。

在 Lambda 架构中，每层都有自己所肩负的任务。

批处理层 存储管理主数据集（不可变的数据集）和预先批处理计算好的视图。

批处理层 使用可处理大量数据的分布式处理系统预先计算结果。它通过处理所有的已有历史数据来实现数据的准确性。这意味着它是基于完整的数据集来重新计算的，能够修复任何错误，然后更新现有的数据视图。输出通常存储在只读数据库中，更新则完全取代现有的预先计算好的视图。

速度处理层 会实时处理新来的大数据。

速度层 通过提供最新数据的实时视图来最小化延迟。速度层所生成的数据视图可能不如批处理层最终生成的视图那样准确或完整，但它们几乎在收到数据后立即可用。而当同样的数据在批处理层处理完成后，在速度层的数据就可以被替代掉了。

Lambda 架构问题：

虽然 Lambda 架构使用起来十分灵活，并且可以适用于很多的应用场景，但在实际应用的时候，Lambda 架构也存在着一些不足，主要表现在它的维护很复杂。

（1）同样的需求需要开发两套一样的代码：这是 Lambda 架构最大的问题，两套代码不仅仅意味着开发困难（同样的需求，一个在批处理引擎上实现，一个在流处理引擎上实现，还要分别构造数据测试保证两者结果一致），后期维护更加困难，比如需求变更后需要分别更改两套代码，独立测试结果，且两个作业需要同步上线。

（2）资源占用增多：同样的逻辑计算两次，整体资源占用会增多（多出实时计算这部分）

# 4 Kappa 架构

Lambda 架构虽然满足了实时的需求，但带来了更多的开发与运维工作，其架构背景是流处理引擎还不完善，流处理的结果只作为临时的、近似的值提供参考。后来随着 Flink 等流处理引擎的出现，流处理技术很成熟了，这时为了解决两套代码的问题，LickedIn 的 Jay Kreps 提出了 Kappa 架构。

Kappa 架构可以认为是 Lambda 架构的简化版（只要移除 lambda 架构中的批处理部分即可）。

在 Kappa 架构中，需求修改或历史数据重新处理都通过上游重放完成。

Kappa 架构最大的问题是流式重新处理历史的吞吐能力会低于批处理，但这个可以通过增加计算资源来弥补。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd852de4593b4d76909b0035daa77f0c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_20,color_FFFFFF,t_70,g_se,x_16)

Kappa 架构的重新处理过程：

重新处理是人们对 Kappa 架构最担心的点，但实际上并不复杂：

（1）选择一个具有重放功能的、能够保存历史数据并支持多消费者的消息队列，根据需求设置历史数据保存的时长，比如 Kafka，可以保存全部历史数据。

（2）当某个或某些指标有重新处理的需求时，按照新逻辑写一个新作业，然后从上游消息队列的最开始重新消费，把结果写到一个新的下游表中。

（3）当新作业赶上进度后，应用切换数据源，读取 2 中产生的新结果表。

（4）停止老的作业，删除老的结果表。

![在这里插入图片描述](https://img-blog.csdnimg.cn/221dfd5b0bd44beeb7f577e1099943ad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_20,color_FFFFFF,t_70,g_se,x_16)

# 5 Lambda 架构与 Kappa 架构的对比

如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b2d436cbfbb14a1680b4bb98b3f73a29.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_16,color_FFFFFF,t_70,g_se,x_16)

1、在真实的场景中，很多时候并不是完全规范的 `Lambda` 架构或 `Kappa` 架构，`可以是两者的混合`，比如大部分实时指标使用 `Kappa` 架构完成计算，少量关键指标（比如金额相关）使用 `Lambda` 架构用批处理重新计算，增加一次校对过程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5c5a17b655ad42edb2db728e4b4b5459.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_17,color_FFFFFF,t_70,g_se,x_16)

2、`Kappa` 架构并不是中间结果完全不落地，现在很多大数据系统都需要支持机器学习（离线训练），所以`实时中间结果需要落地对应的存储引擎供机器学习使用`，另外有时候还需要对明细数据查询，这种场景也需要把实时明细层写出到对应的引擎中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5a7b734a08364c688f7ade1d6f02c350.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA5Liq5YaZ5rm_55qE56iL5bqP54y_,size_18,color_FFFFFF,t_70,g_se,x_16)