- [Hive_关系建模与维度建模_大数据翻身的博客-CSDN博客_hive维度建模](https://blog.csdn.net/qq_56870570/article/details/119807506?spm=1001.2014.3001.5502)

## 1. OLTP与OLAP

当今的数据处理大致可以分成两大类：联机事务处理OLTP（on-line transaction processing）、联机分析处理OLAP（On-Line Analytical Processing）。OLTP是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。二者的主要区别对比如下表所示。

| 对比属性 | OLTP                       | OLAP                       |
| -------- | -------------------------- | -------------------------- |
| 读特性   | 每次查询只返回少量记录     | 对大量记录进行汇总         |
| 写特性   | 随机、低延时写入用户的输入 | 批量导入                   |
| 使用场景 | 用户，Java EE项目          | 内部分析师，为决策提供支持 |
| 数据表征 | 最新数据状态               | 随时间变化的历史状态       |
| 数据规模 | GB                         | TB到PB                     |

## 2. 关系模型与维度模型总体特征

关系模型严格遵循第三范式（3NF），较松散零碎，物理表数量多，数据冗余程度低。由于数据分布于众多的表中，这些数据可以更为灵活地被应用，功能性较强。主要应用于OLTP系统中，保证数据的一致性以及避免冗余，所以大部分业务系统的表都遵循第三范式。

维度模型主要应用于OLAP系统中，通常以某一个事实表为中心进行表的组织，主要面向业务，特征是可能存在数据的冗余，但是能方便的得到数据。

关系模型虽然冗余少，但是在大规模数据，跨表分析统计查询过程中，会造成多表关联，这会大大降低执行效率。所以通常采用维度模型建模，把相关各种表整理成两种：事实表和维度表。

关系建模和维度建模是两种数据仓库的建模技术。关系建模由Bill Inmon所倡导，维度建模由Ralph Kimball所倡导。

## **3.** **关系建模**

关系建模将复杂的数据抽象为两个概念——实体和关系，并使用规范化的方式表示出来。关系模型如图所示，从图中可以看出，较为松散、零碎，物理表数量多。

### ![img](https://img-blog.csdnimg.cn/20210818111506430.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)

关系模型严格遵循第三范式（3NF），数据冗余程度低，数据的一致性容易得到保证。由于数据分布于众多的表中，查询会相对复杂，在大数据的场景下，查询效率相对较低。

## **4.** **维度建模**

维度模型如图所示，从图中可以看出，模型相对清晰、简洁。

![img](https://img-blog.csdnimg.cn/20210818111541380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)

图 维度模型示意图

维度模型以数据分析作为出发点，不遵循三范式，故数据存在一定的冗余。维度模型面向业务，将业务用事实表和维度表呈现出来。表结构简单，故查询简单，查询效率较高。

## 5. 维度模型分类

在维度建模的基础上又分为三种模型：星型模型、雪花模型、星座模型。

### 5.1 星型模型

![img](https://img-blog.csdnimg.cn/20210722104223191.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)

 星型模型和雪花模型的主要区别在于维度的层级，标准的星型模型维度只有一层，而雪花模型会涉及多级。

### 5.2 雪花模型

![img](https://img-blog.csdnimg.cn/20210722104256750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70) 

雪花模型，比较靠近3NF，但是无法完全遵守，因为遵循3NF的性能成本太高。

### 5.3 星座模型

![img](https://img-blog.csdnimg.cn/20210722104354313.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)

 星座模型与前两种的区别是事实表的数量，星座模型是基于多个事实表。

基本上是很多数据仓库的常态，因为很多数据仓库都是多个事实表的，所以星座不星座只反映是否有多个事实表，他们之间是否共享一些维度表。所以星座模型并不和前两种冲突。

### 5.4 模型的选择

首先就是星座不星座这个只跟数据和需求有关系，跟设计没关系，不用选择。

星型还是雪花，取决于性能优化，还是灵活更优先。

目前实际企业开发中，不会绝对选择一种，根据情况灵活组合，甚至并存（一层维度和多层维度都保存）。但是整体来看，更倾向于维度更少的星型模型。尤其是Hadoop体系，减少Join就是减少Shuffle，性能差距很大。（关系型数据可以依靠强大的主键索引）