- [一种基于 Apache Hive 的元数据智能发现方案 (qq.com)](https://mp.weixin.qq.com/s/gcHoEVDQ8duey2siOVqa4w)

## 一、背景

数据湖的生态体系中，会同时存在多种数据存储格式，如 avro、json、csv、text、parquet、orc 等，用户会根据业务情况选用不同的数据格式。在常规的大数据数仓构建中，企业往往选择以 Hive 为中心构建数仓平台，利用 Hive Metastore（HMS）对这些数据存储格式进行表格式的封装，满足多种计算引擎如 Presto、Spark 等以 SQL 查询数据的需求。

使用 HMS 构建数仓，用户需要明确知道原始数据的结构信息，根据数据结构信息在 Hive 中手动创建表结构。如果原始数据结构信息比较复杂，如 json 的嵌套结构体、或者 parquet 包含较多复杂数据类型，用户建表很容易出错。**那么是否有一种方案，用户只需要指定一个数据的文件路径，自动推断数据结构信息并在 Hive 中创建对应的表结构呢？**

## 二、方案调研

首先调研一下业内类似的商业&开源方案：

### 1、商业产品-AWS Glue

AWS Glue 是亚马逊推出的一款 Serverless 数据集成以及 ETL 商业服务，Glue 兼容 HMS 接口，可以认为是商业版的 HMS。Glue 中的元数据爬取&智能发现功能是业内做的最早的，用户只需要把原始数据上传到 AWS 云存储中，Glue 会定时爬取这些数据信息并自动创建好对应的表结构信息，免去了用户手动建表的繁琐，方便了用户的数据上云。

### 2、开源产品-Apache Spark

Apache Spark 自带了一部分的元数据推断能力，这里做一个 SQL 演示，以下测试都是基于 Spark3，有两种基本思路：

1）思路一：用户可以指定文件路径并创建对应的临时表 temporary view，然后在 Spark 中进行 SQL 操作，但是临时表并不会持久化表结构信息至 HMS，需要再次执行一个 SQL 语句 create table like 把临时表信息复制到 HMS 中：

```
// Spark3会话中创建临时表orctbl，临时表在当前会话有效
CREATE TEMPORARY VIEW orctbl USING ORC OPTIONS (path "hdfs://ns/path/orcfile");
// Spark3中使用using指定格式ORC创建Hive表hiveorctbl，表结构信息持久化至HMS
CREATE TABLE hiveorctbl LIKE orctbl USING ORC;
```

**注意：**Spark3 中也可以使用兼容 Hive 的语法即 create table like view sotred as 语义创建 Hive 表，但是这个语法当前有 bug，stored as 后面指定的格式不生效，我们向 Spark 社区提交了 PR 修复了这个问题，见 **https://issues.apache.org/jira/browse/SPARK-40829**。

2）思路二：思路一的优点在于可以灵活指定最终创建表的存储格式(ORC 或者 Parquet 等)，如果用户不需要指定具体数据格式，只是查询当前数据文件，使用一条 SQL 即可：

```
// Spark3中直接根据路径文件创建Hive表hiveorctbl，表结构信息持久化至HMS
CREATE TABLE hiveorctbl USING ORC OPTIONS (path "hdfs://ns/path/orcfile");
```

综上两种商业&开源方案，Glue 的商业方案应该很香，但是需要收费；Spark 的部分元数据推断能力勉强可用，但用户有时并不希望为了使用元数据发现功能而维护一套 Spark 集群，而且 Spark 还缺少一些类似 AWS Glue 新增分区定时添加能力。我们理想的元数据发现方案是在不维护多余组件的情况下，以纯 SQL 的方式实现元数据智能发现。

既然大数据生态是围绕 Hive 构建数仓，那么我们是否可以基于 Hive 去实现这样一种元数据智能发现方案呢？使用 Hive 发现元数据并持久化到 HMS，多种计算引擎再去查询数据，这是一个最直接有效的方案。接下来我们详细介绍一下怎么基于 Hive 去实现这种方案。

## 三、基于 Hive 的元数据智能发现方案

### 1、根据数据路径创建对应的表结构信息

我们首先考虑的是如何在 Hive 中指定一个数据路径，然后自动推断文件的元数据？调研发现，当前 Hive 社区最新代码中做了类似的功能见**https://issues.apache.org/jira/browse/HIVE-26395**。这个issue实现了一个新的Hive语法CTLF，建表的时候只需要指定一个原始数据文件，就可以在HMS中创建出对应的表结构信息，也就是在Hive中做了一个轻量级的元数据发现功能：

```
CREATE TABLE <tableName> LIKE FILE <fileFormat> 'path to file';
```

这个 issue 实现了最基本的 Hive 元数据发现功能框架，并且实现了 PARQUET 数据的元数据发现能力，用户只需要一条 SQL 就可以基于一个 PARQUET 文件在 HMS 中创建对应的表结构信息：

```
CREATE TABLE testhivetbl LIKE FILE PARQUET '/path/parquetfile';
```

基于这个基本框架，我们在 Hive 中实现了 ORC 的元数据发现能力，并贡献到了社区，见 **https://issues.apache.org/jira/browse/HIVE-26551**。Hive 的这个元数据发现特性开发代码量其实很少，只涉及到 antlr 语法、ddl 语义分析、serde 模块三个地方的非侵入性改造，其中核心改造是在 serde 模块中增加解析原始数据 metadata 的能力，如果还需要实现其他数据格式的元数据发现如 json，仅需在 json 的 serde 模块增加 json 数据解析的逻辑即可。下面一张图简述 CTLF 元数据发现流程：

![图片](https://mmbiz.qpic.cn/mmbiz_png/rSj1QeSoL8senGyRLicm76Qx8e5pKBEngrUQicdyJS3Up1S8IA2gce6VPD8WM4fH5l77q8lBzs5cmjnYQpRB8WwQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图 CTLF 语法元数据发现过程

### 2、定时修复分区表的分区信息

上文我们知道了当前 Hive 可以基于一个数据文件使用 CTLF 语义做轻量级的元数据发现，多种计算引擎进而可以查询发现的表。但是实际应用中业务有很多分区表，而分区表的分区目录是动态增加的，这个元数据发现功能并不能感知到新增加的分区，用户需要手动 msck 修复新的分区，这和我们提出的智能元数据发现方案理念不符，我们希望 Hive 中有像 AWS Glue 中的定时发现元数据功能，定时把新的分区目录添加到了 HMS 元数据中，在 Hive 中做到名副其实的智能元数据发现。

经调研，我们发现 Hive4.0 之后增加了两个有意思的特性，都能够满足我们的分区智能添加的需求。

**1）Automatic partition management**

Hive4.0 的 Automatic partition management 自动分区管理功能，见https://issues.apache.org/jira/browse/HIVE-20707。这个特性在表属性中增加了一个参数discover.partitions，用户设置这个参数控制定时（默认间隔5分钟）添加分区目录至HMS元数据中，同时还可以设置分区的最大生命周期，到期后自动Drop掉该分区，这个特性避免了用户每次手动添加msck分区的繁琐，实现了HMS端轻量级智能分区管理能力。所以上文的CTLF结合这里的自动分区管理功能，我们使用一条SQL就可以完成分区表的元数据智能发现：

```
CREATE TABLE testhivetbl LIKE FILE PARQUET '/path/parquetfile' TBLPROPERTIES ('discover.partitions'='true');
```

**2）Scheduled query**

Hive4.0 的 Scheduled query 定期查询功能，见https://issues.apache.org/jira/browse/HIVE-21884。这个特性其实就是在Hive中实现了类似quartz定时调度功能，用户可以在Hive SQL 指定某个查询定期执行，如下创建 scheduled query，每隔 10 分钟执行一次修复 hive 表 testhivetbl 分区的操作，实现分区智能添加的需求，这里结合上文提到的 CTLF 语句，使用两条 SQL 完成分区表的元数据智能发现：

```
CREATE TABLE testhivetbl LIKE FILE PARQUET '/path/parquetfile';
create scheduled query t_msck cron '0 */10 * * * ? *' as msck repair table testhivetbl;
```

但是这个功能需要用户多一次 SQL 操作创建 scheduled query，而且用户还需要管理这些 scheduled query，如删除或者更新 scheduled query；相比之下 HIVE-20707（https://issues.apache.org/jira/browse/HIVE-20707） 的自动分区管理功能略胜一筹，用户只需要在建表的时候设置表属性 discover.partitions 即可，不需要再进行多余 SQL 操作，更符合我们定义的智能化管理分区需求。

**注意：**HIVE-20707 和 HIVE-21884（https://issues.apache.org/jira/browse/HIVE-21884） 这两个特性本质上都是在 HMS 服务端增加了定时执行多线程机制，如果多个 HMS 同时运行这些后台线程可能会出现锁竞争以及重复运行的问题，Hive4.0 之后增加了 HMS 的 Leader 角色来避免这种问题，见 HIVE-21841（https://issues.apache.org/jira/browse/HIVE-21841）。用户可以通过设置参数 hive.metastore.housekeeping.leader.hostname 指定哪一个 HMS 是 Leader，只有 Leader HMS 才会运行这些后台定时线程：

![图片](https://mmbiz.qpic.cn/mmbiz_png/rSj1QeSoL8senGyRLicm76Qx8e5pKBEngxm8hmicOBuvVQJTlrhDcia3ZC7NibKC9Y5hsevRiaFemgndFmqmVtZB9RA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图 参数指定某个 HMS 为 Leader 并用来运行定时线程

## 四、小结

元数据发现功能能够能够很方便地帮助用户创建表结构信息，加速用户的业务分析流程，尤其在大数据上云的趋势下，元数据发现能力能够助力用户快速洞察多源异构的数据元信息。本文基于 Apache Hive 实现了一种元数据智能发现方案，能够满足用户对原始数据表结构信息自动化创建的基本需求，同时我们还向社区贡献了 ORC 的元数据发现功能，后续我们会继续实现其他数据格式的元数据发现能力，以满足用户对多种数据的元数据发现需求。