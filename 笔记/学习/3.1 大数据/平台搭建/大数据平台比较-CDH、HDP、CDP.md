- [大数据平台比较-CDH、HDP、CDP_AllenGd的博客-CSDN博客_cdh hdp](https://blog.csdn.net/Allenzyg/article/details/108197147)

## [Apache](https://so.csdn.net/so/search?q=Apache&spm=1001.2101.3001.7020) Hadoop与CDH、HDP的比较

一、Hadoop版本综述

不收费的Hadoop版本主要有三个（均是国外厂商），分别是：Apache（最原始的版本，所有发行版均基于这个版本进行改进）、Cloudera版本（Cloudera’s Distribution Including Apache Hadoop，简称CDH）、Hortonworks版本(Hortonworks Data Platform，简称HDP），对于国内而言，绝大多数选择CDH版本，CDH和Apache版本主要区别如下： 

二、社区版与第三方发行版的比较

1.Apache社区版

优点：

  完全开源免费

  社区活跃

  文档、资料详实

缺点：

  \1. 版本管理比较混乱，各种版本层出不穷，很难选择，选择其他生态组件时，比如Hive，Sqoop，Flume，Spark等，需要考虑兼容性问题、版本匹配问题、组件冲突问题、编译问题等。

  \2. 集群安装部署复杂，需要编写大量配置文件，分发到每台节点，容易出错，效率低。

  \3. 集群运维复杂，需要安装第三方软件辅助。

2.第三方发行版（CDH）

优点：

  \1. 版本管理清晰。CDH3、CDH4等。

  \2. 版本更新快。通常情况，比如CDH每个季度会有一个update，每一年会有一个release。

  \3. 集群安装部署简单。提供了部署、安装、配置工具，大大提高了集群部署的效率

  \4. 运维简单。提供了管理、监控、诊断、配置修改的工具，管理配置方便，定位问题快速、准确，使运维工作简单，有效。

缺点：

涉及到厂商锁定的问题。

三、第三方发行版的比较

Cloudera：最成型的发行版本，拥有最多的部署案例。提供强大的部署、管理和监控工具。

Hortonworks：不拥有任何私有（非开源）修改地使用了100%开源Apache Hadoop的唯一提供商。Hortonworks是第一家使用了Apache HCatalog的元数据服务特性的提供商。并且，它们的Stinger开创性地极大地优化了Hive项目。Hortonworks为入门提供了一个非常好的，易于使用的沙盒。Hortonworks开发了很多增强特性并提交至核心主干，这使得Apache Hadoop能够在包括Windows Server和Windows Azure在内的Microsft Windows平台上本地运行。

四、版本选择

当我们选择是否采用某个软件用于开源环境时，通常需要考虑：

（1）是否为开源软件，即是否免费。

（2）是否有稳定版，这个一般软件官方网站会给出说明。

（3） 是否经实践验证，这个可通过检查是否有一些大点的公司已经在生产环境中使用知道。

（4）是否有强大的社区支持，当出现一个问题时，能够通过社区、论坛等网络资源快速获取解决方法。

介绍了Apache Hadoop与CDH、HDP的比较，再来看一下

## CDH5与CDH6对比

1 概述

Cloudera于2018年8月30日正式发布CDH6.0.0，至2019年2月19日，迭代版本为CDH6.1.1，现在CDH最新的版本已经到CDH6.3.3了。CDH6是基于Hadoop3同时包含大量其他组件的大版本更新的发布版本，许多用户考虑到未来CDH的版本主要是基于CDH6，而CDH5会慢慢的停止更新，所以考虑新搭集群使用CDH6，或者想把已有的CDH5集群升级到CDH6。第一个问题就是需要考虑CDH5和CDH6的差别，通过分析一些差异看能否将已有的应用迁移或者直接部署到CDH6，兼容性稳定性是否存在问题等。出于这个目的，本文会从各个方面详细比较CDH5和CDH6的差别，从而让用户能够进行正确的判断并进行相应的选择。以下内容主要基于最新的CDH5.16.1和CDH6.1.1进行比较。最后再次强调，没有最好的技术，也没有最新的技术就是最好的，永远只有最合适的技术。

2 组件版本对比

| ***\*组件\****     | ***\*CDH5.16.1\**** | ***\*CDH6.1.1\**** |
| ------------------ | ------------------- | ------------------ |
| Cloudera Manager   | 5.16.1              | 6.1.1              |
| CDSW               | 1.5                 | 1.5                |
| Cloudera Navigator | 2.15.1              | 6.1.1              |
| Flume-ng           | 1.6.0               | 1.8.0              |
| Hadoop             | 2.6.0               | 3.0.0              |
| HBase              | 1.2.0               | 2.1.0              |
| Hive               | 1.1.0               | 2.1.1              |
| Hue                | 3.9.0               | 3.9.0              |
| Impala             | 2.12.0              | 3.1.0              |
| Oozie              | 4.1.0               | 5.0.0              |
| Pig                | 0.12.0              | 0.17.0             |
| Sentry             | 1.5.1               | 2.1.0              |
| Solr               | 4.10.3              | 7.4.0              |
| Spark              | 1.6.0/2.3.0         | 2.4.0              |
| Sqoop              | 1.4.6               | 1.4.7              |
| Sqoop2             | 1.99.5              | 无                 |
| Zookeeper          | 3.4.5               | 3.4.5              |
| Kafka              | 1.0.1               | 2.0.0              |
| Kudu               | 1.7.0               | 1.8.0              |

3 操作系统支持

| ***\*操作系统\****              | ***\*CDH5.16.1\****                                          | ***\*CDH6.1.1\****                              |
| ------------------------------- | ------------------------------------------------------------ | ----------------------------------------------- |
| RHEL/CentOS/OL with RHCK kernel | 7.6, 7.5, 7.4, 7.3, 7.2, 7.1, 6.10, 6.9 , 6.8, 6.7, 6.6, 6.5, 6.4, 5.11, 5.10, 5.7 | 7.6, 7.5, 7.4, 7.3, 7.2, 6.10, 6.9 , 6.8        |
| Oracle Linux (OL)               | 7.5, 7.4, 7.3, 7.2, 7.1 (UEK default), 6.9, 6.8 (UEK R2, R4), 6.7, 6.6, 6.5 (UEK R2, R3), 6.4 (UEK R2), 5.11, 5.10, 5.7 (UEK R2) | 7.4, 7.3, 7.2 (UEK default), 6.10 (UEK default) |
| SLES                            | 12 SP3, 12 SP2, 12 SP1, 11 SP4, 11 SP3                       | 12 SP3, 12 SP2                                  |
| Ubuntu                          | 16.04 LTS (Xenial), 14.04 LTS (Trusty)                       | 16.04 LTS (Xenial)                              |
| Debian                          | 8.9, 8.4, 8.2 (Jessie), 7.8, 7.1, 7.0 (Wheezy)               | 不支持                                          |

4 元数据库支持

| ***\*数据库\**** | ***\*CDH5.16.1\****                             | ***\*CDH6.1.1\****            |
| ---------------- | ----------------------------------------------- | ----------------------------- |
| MySQL            | 5.1, 5.5, 5.6, 5.7                              | 5.7                           |
| MariaDB          | 5.5, 10.0                                       | 5.5, 10.0                     |
| PostgreSQL       | 8.1, 8.3, 8.4, 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 10 | 8.4, 9.2, 9.4, 9.5, 9.6, 10.x |
| Oracle           | 11g R2, 12c R1, 12.2                            | 12.2                          |

5 JDK支持

| ***\*JDK\**** | ***\*CDH5.16.1\****                                          | ***\*CDH6.1.1\****                                           |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| JDK7          | 1.7u55, 1.7u67, 1.7u75, 1.7u80                               | 不支持                                                       |
| JDK8          | 1.8u31, 1.8u74, 1.8u91, 1.8u102, 1.8u111, 1.8u121, 1.8u131, 1.8u144, 1.8u162, 1.8u181 | 1.8u31, 1.8u74, 1.8u91, 1.8u102, 1.8u111, 1.8u121, 1.8u131, 1.8u141, 1.8u162, 1.8u181 |
| OpenJDK       | 1.8u181(CDH5.16之前的版本不支持OpenJDK)                      | 1.8u181                                                      |

6 单用户模式

我们一般在安装CDH时，都是使用root或具有sudo权限的用户安装的，如果大家有注意会发现用于管理每台主机上的Hadoop进程的Cloudera Manager Agent服务（cloudera-scm-agent）是以root用户身份运行的。但是在有些企业，运维部门有严格的要求，需要CDH使用自己的用户来管理即不能随便使用root，比如要求cloudera-scm-agent服务以其他用户进行启停和管理。从Cloudera Manager 5.3开始Cloudera提供了single user mode，来满足这种要求。在单用户模式下，Cloudera Manager Agent和所有由Cloudera Manager管理的服务进程都可以由配置的用户来启动。单用户模式优先考虑Hadoop与系统其余进程之间的隔离，而不是Hadoop进程之间的隔离。对于Cloudera Manager的部署，单用户模式是全局的，适用于由该Cloudera Manager管理的所有集群。默认情况下，单用户是cloudera-scm。

从CDH6开始，不再支持单用户模式，安装必须使用root或具有sudo权限的用户安装。

7 总结

CDH6相对于CDH5是一次各个组件的大版本升级，要理解大版本更新，这里先解释一下Hadoop相关组件的版本说明。比如版本号是x.y.z，z代表的一般是Maintenance Version或Patch Version，这种版本升级主要修复bug，不改动API，也不涉及新的功能，y代表的是Minor Version，这种版本升级主要是增加新的功能和新的API，x代表的是Major Version，往往会新增全新的功能，甚至改动API。那么CDH5到CDH6就是一次Major Version的升级，新增了很多功能，API的更改有可能导致你旧有的程序的不兼容而需要修改或者重新开发，但根据软件开发迭代的原则，往往新版本会向下兼容一段时间，你只需要经过严格的测试，在一段时间内应用开发的修改理论不会太大，但还是要结合实际情况，进行严格的测试后再行判断和评估。

CDH5现在依旧是市场上用的最多，推得最广的版本，客户认可度也较高。同时，CDH5经过快5年的迭代也是目前最成熟和稳定的版本，各个组件跟随着社区版的更新迭代，已经把各个组件的bug修复得很完善。另外，无论是国外还是国内支持案例也多，国内可以找到很多相关的实施案例供参考，可以节省运维人力、物力成本。最后CDH5的周边生态圈支持也比较丰富，无论是开源还是商业产品，ETL工具，调度工具，报表工具，BI工具等。你如果现在选择CDH5或者已经在用了，Cloudera也官宣了依旧会再支持三年。

CDH6的更新较大，很多组件也都是大版本更新，引入了很多激动人心的新功能，同时各个组件也修复了大量已知的问题和安全漏洞。比如HDFS的[纠删码](https://blog.csdn.net/Allenzyg/article/details/106355770)用于冷数据降低存储成本又保证了数据的可用性，NameNode和YARN的联邦解决大规模集群的性能瓶颈问题，YARN引入GPU支持，未来会引入docker支持等。长远来看，从Hadoop2升级到Hadoop3或者从CDH5升级到CDH6是必须的，因为无论是社区还是Cloudera从现在开始开始都会将开发重心转移到Hadoop3或CDH6上，而CDH5则主要以维护和修复bug为主。

建议：

1.如果你是新搭集群，并且规模不太大(50个节点以内)，基于Hadoop的应用也都是新开发，CDH6是一个不错的选择，毕竟避免了以后还要考虑从CDH5升级到CDH6的麻烦，同时还需要进行一次应用测试和迁移。未来迁移到CDH和HDP的合并版本CDP也会更加容易。

2.如果你已经有CDH5集群，作为生产系统各种应用已经稳定运行了较长的时间，如果不是因为功能或性能原因必须要用新版本，现阶段可以暂时不考虑升级，继续观望一段时间。而你一旦决定升级，需要谨慎和细致的对待，包括你要考虑OS，JDK，元数据库等的升级，已有各种应用比如Hive/Impala/Spark SQL，MapReduce/Spark代码，脚本，Python/R算法工程等在CDH6进行测试，同时周边的一些工具对接比如ETL，调度，报表，BI工具等也需要进行对接测试，一切测试通过以后规划好合理的停机时间，然后进行升级。

注：以上总结分析和建议不具备通用性，仅供参考，如果您在选择CDH版本时存在问题，请联系Cloudera销售代表和技术支持结合您的实际情况进行合理的分析和建议。

有的CDH5版本已经停止支持了。

停止支持的意思是不再更新和修复bug，厂商不再向这个版本的用户提供技术支持。CDH5到今年年底全部停止支持。

| 产品列表（版本）         | 停止支持时间 |
| ------------------------ | ------------ |
| Cloudera Enterprise 6.3  | 2022年3月    |
| Cloudera Enterprise 6.2  | 2022年3月    |
| Cloudera Enterprise 6.1  | 2021年12月   |
| Cloudera Enterprise 6.0  | 2021年8月    |
| Cloudera Enterprise 5.16 | 2020年12月   |
| Cloudera Enterprise 5.15 | 2020年12月   |
| Cloudera Enterprise 5.14 | 2020年12月   |
| Cloudera Enterprise 5.13 | 2020年10月   |
| Cloudera Enterprise 5.12 | 2020年7月    |
| Cloudera Enterprise 5.11 | 已停止       |
| Cloudera Enterprise 5.10 | 已停止       |
| Cloudera Enterprise 5.9  | 已停止       |

## CDH用户及开发者应当关注的CDP组件变化

在Cloudera和Hortonworks合并后，Cloudera公司推出了新一代的数据平台产品CDP Data Center（以下简称为CDP），在2019年11月30日已经正式GA。

CDP的版本号延续了之前CDH的版本号，从7.0开始，目前最新的版本号为7.0.3.0。

那么CDP对比之前的Cloudera Enterprise Data Hub（CDH企业版）与HDPEnterprise Plus（HDP企业版）到底在组件上发生了哪些变化呢？

由于HDP在国内市场上的市场占有量很小，大部分公司都是采用CDH，所以对于HDP带来的一些东西，使用CDH的用户和开发人员会比较陌生，下面带大家详细的了解一下CDP中的组件一些变化，也方便大家为在以后的学习做好准备。

**1 CDP、CDH、HDP中都包含的部分**

Apache Hadoop（HDFS/YARN/MR）

Apache HBase

Apache Hive

Apache Oozie

Apache Spark

Apache Sqoop

Apache Zookeeper

Apache Parquet

Apache Phoenix（*CDH中需要额外安装）

基本上只是版本做了一定的升级。如果你之前是使用CDH的用户，那么其中值得一提的是Hive的版本在CDP中使用的是3.1.2，对比CDH6来说是很大的升级（CDH6中Hive为2.1.1）。之前Cloudera在组件的选择上是比较保守的，对比之下Hortonworks的策略会比较激进（HDP已经上了Hive3），会更贴近社区最新的版本。

Hive 3的几个重要新特性：

默认执行引擎变更为TEZ

默认开启ACID支持，用于支持事务

LLAP支持，用于支持秒级、毫秒级的实时查询

客户端方面彻底废弃掉了Hive CLI，这也意味着beeline成为唯一选择

**2 CDH中存在，CDP还在准备中的部分**

Apache Accumulo

Navigator Encrypt

Key HSM

实际实施中这些用的都不多，而且迟早会加入到CDP中，不需要太过关注

**3 CDP、CDH中都包含的部分**

Apache Avro

Cloudera Manager

Hue

Apache Impala

Key Trustee Server

Apache Kudu

Apache Solr

Apache Kafka（*CDH中需要额外安装）

**4 HDP中存在，CDP还在准备中的部分**

Apache Druid

Apache Knox

Apache Livy

Ranger KMS

Apache Zeppelin

其中最需要关注的就是Apache Druid。Apache Druid是一款实时大数据分析引擎，注意它与阿里巴巴出品的一款数据库连接池Druid恰好重名，但实际是两款不同的软件，两者之间并没有什么关系。

Apache Druid能快速地对海量数据进行实时存储和多维度实时分析。在快速查询、水平扩展、实时数据摄入和分析这三方面都有良好的支持。本质上来说就是一款时序数据库，能很方便的对时间驱动型的数据进行快速的分析查询。

需要注意的是为了达到高性能，Druid牺牲了很多方面的特性，例如并不支持完整的SQL语义特性（只提供有限制的join支持）。所以Druid并非是Hive或者Impala的替代品，它们之间是一种相互增强的关系，例如以下场景：选用Hive或Impala用于构建离线数据仓库或数据集市，然后在这个基础上，将需要进行多维分析查询的数据放入Druid进行加工，来给多维分析类系统提供数据接口。[Apache Druid vs Kudu](https://blog.csdn.net/Allenzyg/article/details/109999034)

**5 CDP、HDP中都包含的部分**

Apache Atlas

Apache Ranger

Apache ORC

Apache Tez

在CDP中，有两个之前CDH的组件被去除，一个是Apache Sentry，用于安全授权；一个是Cloudera Navigator，用于元数据管理与审计。

被去除的原因在于在CDP中它们的功能已经被其他组件所替代，CDP将会使用以Ranger+Knox+Atlas为中心的安全授权与审计方案。对于使用CDH的用户来说，这部分的变化是最大的。

目前CDH 6.3的EoS Date（支持结束日期）为2022年3月，接下来两年内基于CDH6的实施项目仍然是主流选择，基于Sentry的方案仍然会在日常的项目实施中使用到。所以如果你对Hadoop安全体系这块不熟悉，也不能说就干脆跳过Sentry不学了。而且可预见的是将来肯定会有关于现有的CDH集群升级时，涉及到权限迁移这一块的问题，所以Sentry和Ranger两者都需要了解。

而Apache ORC虽然之前在CDH中其实一直能够使用，但Impala本身对ORC格式的支持非常差，在CDH中无论是在Impala还是Hive中使用ORC格式，都一直没有受到CDH的支持和推荐（CDH都推荐使用Parquet格式）。其实这本身有点商业策略考量的意味：在未合并前，社区中Parquet项目由Cloudera主导，而ORC项目由Hortonworks主导。其实这两种格式在整个Hadoop生态圈中都有着广泛的应用，合并之后CDP中自然是两者都需要提供官方支持了。

Tez也基本上处于同样的理由（商业策略考量），所以之前CDH中的Hive一直是无法直接使用Tez的（选项页面直接选择的话，只能选择MR和Spark作为引擎）。现在合并后并不存在这个商业策略考量的问题了。

**6 CDP全新加入的部分**

Apache Ozone（技术预览版）

Ozone是一个可伸缩的、冗余的、分布式的Hadoop对象存储。

基于Apache Spark、YARN和Hive等框架的应用程序无需任何修改就可以直接在Ozone上工作。

Ozone建立在具有高可用性的复制块存储层上，这个存储层被称为Hadoop Distributed Data Store(HDDS)

简单来说，Ozone就是为了解决HDFS现存的受到NameNode严重制约的超大规模的扩展能力问题以及存储小文件的问题而设计的，Ozone被设计成支持数百亿（在未来甚至更多）的文件和块。

目前Ozone还处于技术预览版阶段，如果有时间做一些基本了解即可，目前还不会有用于实际生产环境的可能性。

## CDP7有感

很多人都知道，Cloudera和Hortonworks两大hadoop服务商在2018年国庆节期间合并了，当时的新闻：[点击](https://www.cnbc.com/2018/10/03/cloudera-and-hortonworks-announce-all-stock-merger.html?from=timeline)

在使用HDP集群那个年代，hive on tez 配合orc存储和SparkSQL，已经可以在大数据平台实现现在很多供应商所说的HTAP了，当时使用sqoop+pig+tez+hive+oozie+falcon跑批，用phoenix+hbase+kafka+spark跑流任务。那时候觉得orc+hive+hplsql真好用，比vertica好用太多太多。

接触CDH集群，第一次用到impala，任务量一上去，经常会爆内存，后来还专门了解了impala的优化。impala对于orc格式仅能做到只读，不过Cloudera也有kudu作为他们介于OLAP和OLTP之间的产物供使用，实际使用效果也不错。

两家合并后半年的时间 CDF就出现了，结果发现，这不就是[NIFI](https://blog.csdn.net/Allenzyg/article/details/105763270)嘛

这次CDP7，不会像从CDH5升级CDH6一样令人开心，CDH6目前使用感受不错，偶尔有小BUG也还能忍受。 CDH5里很多组件版本太老，很多企业在线上使用CDH5好多年可能都不敢升级了，当然也可以自己安装高版本的应用，但不享受厂商的售后服务。CDP7相比于CDH6整体停下来就是想告诉我们一句话 "上云！"

一直在围绕"云" 大谈特谈，支持公有云，私有云，本地部署方式。

国外有AWS,Azure,Google，国内有阿里，腾讯，华为，很遗憾，CDP7目前只支持AWS和Azure，跟国内云厂商还没谈好合作，所以国内公有云短期内是没别惦记了。

Ozone要等到CDP7.2才能支持。

再来说说私有云，CDP7的私有云的前提是先部署CDP7的数据中心版本，也就是跟CDH6类似的安装部署方式，本地部署。数据存储方式还是HDFS，暂不支持Ozone对象存储。然后私有云部署方式相比于数据中心版有什么好处呢，是DevOps还是存储计算分离？ 消耗更大的网络带宽换来的优势是否值得还需要再好好思考一番。

最后就是数据中心版本CDP-DC， 我们也可以叫他CDH7，本质上就是对CDH6进行了组件的升级，将部分CDH的组件替换成HDP的组件，比如sentry变成了ranger，navigator变成了atlas， hive2升级到hive3。 其他的组件都大差不差，和CDH6.3相比区别不大。

升级的时候，sentry的权限如何同步到ranger，是否有风险？ 目前仅支持从CDH5升级到CDP7，还不支持CDH6升级CDP7。

HDFS的小文件问题被大家诟病，Ozone对象存储也是千呼万唤始出来，然而有点晚了。云厂商发展的太快，包装好的服务拿来直接就能用，CDH和HDP斗了那么久，各自都造了不少轮子，合并之后也开始把功能重叠的做替换，好想感叹一句"大数据的轮子咋就那么多呢？" DeltaLake、Hudi、Iceberg还没整明白呢，那边一下子冒出好多HTAP的数据库厂商。咋那么多功能重叠的组件呢，唉！ 愿hadoop生态越来越好越来越好纯净。

最关键的一点：CDP的组件代码在github上找不到，是不再开源了，CDP7以后就没有社区版了。