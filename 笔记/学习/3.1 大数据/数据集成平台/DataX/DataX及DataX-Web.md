- [大数据Hadoop之——数据同步工具DataX](https://juejin.cn/post/7100939690898882573#heading-10)
- [数据采集工具-DataX ](https://zhuanlan.zhihu.com/p/435622257)
- [datax详细介绍及使用](https://cloud.tencent.com/developer/article/1954484)

## 一、概述

> DataX 是阿里云DataWorks数据集成的开源版本，在阿里巴巴集团内被广泛使用的离线数据同步工具/平台。DataX 实现了包括 MySQL、Oracle、OceanBase、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、Hologres、DRDS 等各种异构数据源之间高效的数据同步功能。

Gitee：[github.com/alibaba/Dat…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Falibaba%2FDataX)

GitHub地址：[github.com/alibaba/Dat…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Falibaba%2FDataX)

文档：[github.com/alibaba/Dat…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Falibaba%2FDataX%2Fblob%2Fmaster%2Fintroduction.md)

> DataX 是一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。

![img](https://img-blog.csdnimg.cn/img_convert/48d21111d66f26a9d243b088f1cd6efe.png)

- 为了解决异构数据源同步问题，DataX将复杂的网状的同步链路变成了星型数据链路，DataX作为中间传输载体负责连接各种数据源。当需要接入一个新的数据源的时候，只需要将此数据源对接到DataX，便能跟已有的数据源做到无缝数据同步。
- DataX在阿里巴巴集团内被广泛使用，承担了所有大数据的离线同步业务，并已持续稳定运行了6年之久。目前每天完成同步8w多道作业，每日传输数据量超过300TB。

## 二、DataX3.0框架设计

> DataX本身作为离线数据同步框架，采用Framework + plugin架构构建。将数据源读取和写入抽象成为Reader/Writer插件，纳入到整个同步框架中。

![img](https://img-blog.csdnimg.cn/img_convert/7d9782d2adfd512f491a0a68d8c3fa93.png)

- **Reader**：Reader为数据采集模块，**负责采集数据源的数据**，将数据发送给Framework。
- **Writer**： Writer为数据写入模块，**负责不断向Framework取数据**，并将数据写入到目的端。
- **Framework**：Framework用于连接reader和writer，作为两者的数据传输通道，并处理缓冲，流控，并发，数据转换等核心技术问题。

### 2.1 核心流程架构图

![image.png](https://img-blog.csdnimg.cn/img_convert/af2b9fe64d9efc99c0b98ae1d2ce2776.webp?x-oss-process=image/format,png)

### 2.2 核心流程

1. job 是 Datax 一次任务的统称，DataX JobContainer 是一个运行 job 的容器，是整个同步任务的管理控制中心，承担了插件初始化，数据清理、 数据检测、任务切分、TaskGroup 管理，任务调度，监控，回收等功能。
2. DataX Job 启动后，会根据不同的源端切分策略，将 Job 切分成多个小的 Task(子任务)，实质上是在切分配置文件，以便于并发执行。Task 便是 DataX 同步任务的基础单位，每一个 Task 都会负责一部分数据的同步工作。
3. 切分后的 task，会根据并发要求，通过 schedule 方法重新组合成 TaskGroup，TaskGroup 线程由一个线程池维护并监控，一个 TaskGroup 默 认并发 5 个 Task。
4. 每一个 Task 都由 TaskGroup 负责启动，Task 启动后，会启动读写两 个线程，并通过 Record 类作为媒介，Reader 不断地读出数据，并往传输中 转站 Channel 中存入信息，还 Writer 则负责从 Channel 中读出 Record 信 息，存入目标数据源。
5. DataX 作业运行时，JobContainer 会监控各个 TaskGroup 模块任务， 直到所有任务完成，并记录日志，当有都成功后会返回 0，不然会有完整的报错机制，异常退出返回非 0。

## 三、DataX3.0架构

> DataX 3.0 开源版本支持单机多线程模式完成同步作业运行，本小节按一个DataX作业生命周期的时序图，从整体架构设计非常简要说明DataX各个模块相互关系。

![img](https://img-blog.csdnimg.cn/img_convert/341b3e8befa17f558640a3fca7c6d607.png)

### 3.1 核心模块介绍

- **DataX完成单个数据同步的作业，我们称之为Job**，DataX接受到一个Job之后，将启动一个进程来完成整个作业同步过程。DataX Job模块是单个作业的中枢管理节点，承担了数据清理、子任务切分(将单一作业计算转化为多个子Task)、TaskGroup管理等功能。
- DataXJob启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。**Task便是DataX作业的最小单元**，每一个Task都会负责一部分数据的同步工作。
- 切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，**默认单个任务组的并发数量为5**。
- 每一个Task都由TaskGroup负责启动，Task启动后，会固定启动`Reader—>Channel—>Writer`的线程来完成任务同步工作。
- DataX作业运行起来之后， Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出，进程退出值非0

### 3.2 DataX调度流程

举例来说，用户提交了一个DataX作业，并且配置了20个并发，目的是将一个100张分表的mysql数据同步到`odps（Open Data Processing Service：开发数据处理服务）`里面。 DataX的调度决策思路是：

- DataXJob根据分库分表切分成了100个Task。
- 根据20个并发，DataX计算共需要分配4个TaskGroup。
- 4个TaskGroup平分切分好的100个Task，每一个TaskGroup负责以5个并发共计运行25个Task。

### 3.3 DataX3.0插件体系

经过几年积累，DataX目前已经有了比较全面的插件体系，主流的RDBMS数据库、NOSQL、大数据计算系统都已经接入。DataX目前支持数据如下：

| 类型               | 数据源                          | Reader(读) | Writer(写) | 文档    |
| :----------------- | :------------------------------ | :--------- | :--------- | :------ |
| RDBMS 关系型数据库 | MySQL                           | √          | √          | 读 、写 |
|                    | Oracle                          | √          | √          | 读 、写 |
|                    | SQLServer                       | √          | √          | 读 、写 |
|                    | PostgreSQL                      | √          | √          | 读 、写 |
|                    | DRDS                            | √          | √          | 读 、写 |
|                    | 达梦                            | √          | √          | 读 、写 |
|                    | 通用RDBMS(支持所有关系型数据库) | √          | √          | 读 、写 |
| 阿里云数仓数据存储 | ODPS                            | √          | √          | 读 、写 |
|                    | ADS                             |            | √          | 写      |
|                    | OSS                             | √          | √          | 读 、写 |
|                    | OCS                             | √          | √          | 读 、写 |
| NoSQL数据存储      | OTS                             | √          | √          | 读 、写 |
|                    | Hbase0.94                       | √          | √          | 读 、写 |
|                    | Hbase1.1                        | √          | √          | 读 、写 |
|                    | MongoDB                         | √          | √          | 读 、写 |
|                    | Hive                            | √          | √          | 读 、写 |
| 无结构化数据存储   | TxtFile                         | √          | √          | 读 、写 |
|                    | FTP                             | √          | √          | 读 、写 |
|                    | HDFS                            | √          | √          | 读 、写 |
|                    | Elasticsearch                   |            | √          | 写      |

DataX Framework提供了简单的接口与插件交互，提供简单的插件接入机制，只需要任意加上一种插件，就能无缝对接其他数据源。详情请看：[DataX数据源指南](https://github.com/alibaba/DataX/wiki/DataX-all-data-channels)

## 四、环境部署

### 1）下载

```bash
$ mkdir -p /opt/bigdata/hadoop/software/datax ; cd /opt/bigdata/hadoop/software/datax
$ wget http://datax-opensource.oss-cn-hangzhou.aliyuncs.com/datax.tar.gz
$ tar -xf datax.tar.gz -C /opt/bigdata/hadoop/server/
```

### 2）设置环境变量

```bash
$ cd /opt/bigdata/hadoop/server/
$ vi /etc/profile
export DATAX_HOME=/opt/bigdata/hadoop/server/datax
export PATH=$DATAX_HOME/bin:$PATH
$ source /etc/profile
```

## 五、DataX优化

- [DataX使用、同步HDFS数据到MySQL案例、DataX优化](https://blog.csdn.net/Redamancy06/article/details/128048258)
- [dataX调优](https://www.cnblogs.com/hit-zb/p/10940849.html)

### 5.1 DataX本身的参数调优

全局：

```json
{
    "core":{
        "transport":{
            "channel":{
                "speed":{
                    "channel": 2, ## 此处为数据导入的并发度，建议根据服务器硬件进行调优
                    "record":-1, ##此处解除对读取行数的限制
                    "byte":-1, ##此处解除对字节的限制
                    "batchSize":2048 ##每次读取batch的大小
                }
            }
        }
    },
    "job":{
        ...
    }
    }
```

局部：

```json
"setting": {
            "speed": {
                "channel": 2,
                "record":-1,
                "byte":-1,
                "batchSize":2048
            }
        }
    }
}

# channel增大，为防止OOM，需要修改datax工具的datax.py文件。
# 如下所示，可根据任务机的实际配置，提升-Xms与-Xmx，来防止OOM。
# channel并不是越大越好，过分大反而会影响宿主机的性能。
DEFAULT_JVM = "-Xms1g -Xmx1g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=%s/log" % (DATAX_HOME)
```

### 5.2 速度控制

DataX3.0提供了包括通道(并发)、记录流、字节流三种流控模式，可以随意控制你的作业速度，让你的作业在数据库可以承受的范围内达到最佳的同步速度。

关键优化参数如下：

| 参数                                | 说明                                                |
| ----------------------------------- | --------------------------------------------------- |
| job.setting.speed.channel           | 并发数                                              |
| job.setting.speed.record            | 总record限速 (tps每秒处理的条数）                   |
| job.setting.speed.byte              | 总byte限速 (bps每秒处理的字节数)                    |
| core.transport.channel.speed.record | 单个channel的record限速，默认值为10000（10000条/s） |
| core.transport.channel.speed.byte   | 单个channel的byte限速，默认值1024*1024（1M/s）      |

注意事项：

1.若配置了总record限速，则必须配置单个channel的record限速

2.若配置了总byte限速，则必须配置单个channe的byte限速

3.若配置了总record限速和总byte限速，channel并发数参数就会失效。因为配置了总record限速和总byte限速之后，实际channel并发数是通过计算得到的：

计算公式为:

min(总byte限速/单个channel的byte限速，总record限速/单个channel的record限速)

配置示例：

```powershell
{
    "core": {
        "transport": {
            "channel": {
                "speed": {
                    "byte": 1048576 //单个channel byte限速1M/s
                }
            }
        }
    },
    "job": {
        "setting": {
            "speed": {
                "byte" : 5242880 //总byte限速5M/s
            }
        },
        ...
    }
}
```

### 5.3 内存调整

当提升DataX Job内Channel并发数时，内存的占用会显著增加，因为DataX作为数据交换通道，在内存中会缓存较多的数据。例如Channel中会有一个Buffer，作为临时的数据交换的缓冲区，而在部分Reader和Writer的中，也会存在一些Buffer，为了防止OOM等错误，需调大JVM的堆内存。

建议将内存设置为4G或者8G，这个也可以根据实际情况来调整。

调整JVM xms xmx参数的两种方式：一种是直接更改datax.py脚本；另一种是在启动的时候，加上对应的参数，如下：

```powershell
python datax/bin/datax.py --jvm="-Xms8G -Xmx8G" /path/to/your/job.json
```

## 六、DataX-Web

GitHub地址：[WeiYe-Jing/datax-web](https://github.com/WeiYe-Jing/datax-web)

前端地址：https://github.com/WeiYe-Jing/datax-web-ui

作者开源中国博客地址：https://segmentfault.com/u/weiye_jing/articles

![img](https://img-blog.csdnimg.cn/img_convert/d2832ddd2268edd305fcebb84fed9e64.png)

### 6.1 DataX-web安装

- [datax + datax_web避坑指南 ](https://www.cnblogs.com/zsf-note/p/15727339.html)
- [CentOS7安装DataX和datax-web](https://www.cnblogs.com/simple-young/p/16295144.html)

### 6.2 DataX-Web使用

- [Datax-web 源码阅读记录](https://blog.csdn.net/qq_39945938/article/details/118972382)

#### 6.2.1 前端访问

部署完成后，在浏览器中输入 http://ip:port/index.html 就可以访问对应的主界面（ip为datax-admin部署所在服务器ip,port为为datax-admin 指定的运行端口9527），初始账号：admin，密码：123456。

#### 6.2.2 datax-web API

datax-web部署成功后，可以了解datax-web API相关内容，网址: http://ip:port/doc.html

#### 6.2.3 路由策略

当执行器集群部署时，提供丰富的路由策略，包括:

- FIRST（第一个）：固定选择第一个机器； 
- LAST（最后一个）：固定选择最后一个机器； 
- ROUND（轮询）：依次分配任务； 
- RANDOM（随机）：随机选择在线的机器； 
- CONSISTENT_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。 
- LEAST_FREQUENTLY_USED（最不经常使用）：使用频率最低的机器优先被选举； 
- LEAST_RECENTLY_USED（最近最久未使用）：最久为使用的机器优先被选举； 
- FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度； 
- BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；

阻塞处理策略：调度过于密集执行器来不及处理时的处理策略

- 单机串行：调度请求进入单机执行器后，调度请求进入FIFO队列并以串行方式运行；
- 丢弃后续调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，本次请求将会被丢弃并标记为失败；
- 覆盖之前调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，将会终止运行中的调度任务并清空队列，然后运行本地调度任务；

增量增新建议将阻塞策略设置为丢弃后续调度或者单机串行

设置单机串行时应该注意合理设置重试次数(失败重试的次数* 每次执行时间< 任务的调度周期)，重试的次数如果设置的过多会导致数据重复，例如任务30秒执行一次，每次执行时间需要20秒，设置重试三次，如果任务失败了，第一个重试的时间段为1577755680-1577756680，重试任务没结束，新任务又开启，那新任务的时间段会是1577755680-1577758680

### 6.3 DataX-Web打包部署

- [datax-web在windows环境idea中模块化打包部署操作步骤](https://blog.csdn.net/u012874077/article/details/116148000)

#### 6.3.1 pom.xml修改

分别在datax-admin和datax-executor下面的pom.xml文件中添加

```html
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

#### 6.3.2 将静态资源移入jar包中

将datax-admin下的所有配置资源拷贝进datax-admin-2.1.2.jar的相关目录中。具体如下：

 1）将idea中的datax-admin下的classes下的配置文件application.yml、bootstrap.properties、logback.xml拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes下。

 2）将datax-admin下的target/classes/i8n下的message.properties和message_en.properties拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes\i18n下

3）将将datax-admin下的target/classes/mybatis-mapper下的所有xml文件拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes\mybatis-mapper下

4）将idea中的datax-executor编译生成后的classes下的配置文件application.yml、logback.xml拷贝进datax-executor-2.1.2.jar下的BOOT-INF\classes下。

### 6.4 参考资料

- [Datax3.0+DataX-Web打造分布式可视化ETL系统](https://zhuanlan.zhihu.com/p/554518685)
- [linux搭建datax、datax-web](https://www.cnblogs.com/zuouncle/p/16087552.html)
- [ETL工具datax任务构建可视化管理](https://www.helloworld.net/p/5256032033)
- [Datax-web 集群化部署使用（图文教程超详细）](https://www.jianshu.com/p/6afc8eaf2f9a)
- [Datax-web入门配置与启动](https://www.cnblogs.com/nnnnmmmm/p/16138276.html)
- [Datax-web的入门使用](https://www.cnblogs.com/nnnnmmmm/p/16140466.html)
- [数据同步工具—DataX—Web部署使用](https://blog.51cto.com/u_12279910/5736280)
- [datax及datax-web部署及填坑记录](https://juejin.cn/post/6998334199757275149)

## 七、DataX和Sqoop、Kettle等的比较

- [【知识】ETL大数据集成工具Sqoop、dataX、Kettle、Canal、StreamSets大比拼](https://www.jianshu.com/p/23ec3a386dc9)
- [六种 主流ETL 工具的比较(DataPipeline，Kettle，Talend，Informatica，Datax ，Oracle Goldengate) ](https://www.cnblogs.com/DataPipeline2018/p/11131723.html)

以下对开源的Sqoop、dataX、Kettle、Canal、StreamSetst进行简单梳理比较，通过分析，建议优先DataX更优。

| 比较维度\产品 |                | DataPipeline                                                 | kettle                                                       | Oracle Goldengate                                            | informatica                                                  | talend                                                       | DataX                                                        |
| ------------- | -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 设计及架构    | 适用场景       | 主要用于各类数据融合、数据交换场景，专为超大数据量、高度复杂的数据链路设计的灵活、可扩展的数据交换平台 | 面向数据仓库建模传统ETL工具                                  | 主要用于数据备份、容灾                                       | 面向数据仓库建模传统ETL工具                                  | 面向数据仓库建模传统ETL工具                                  | 面向数据仓库建模传统ETL工具                                  |
|               | 使用方式       | 全流程图形化界面，应用端采用B/S架构，Cloud Native为云而生，所有操作在浏览器内就可以完成，不需要额外的开发和生产发布 | C/S客户端模式，开发和生产环境需要独立部署，任务的编写、调试、修改都在本地，需要发布到生产环境，线上生产环境没有界面，需要通过日志来调试、debug，效率低，费时费力 | 没有图形化的界面，操作皆为命令行方式，可配置能力差           | C/S客户端模式，开发和生产环境需要独立部署，任务的编写、调试、修改都在本地，需要发布到生产环境；学习成本较高，一般需要受过专业培训的工程师才能使用； | C/S客户端模式，开发和生产环境需要独立部署，任务的编写、调试、修改都在本地，需要发布到生产环境； | DataX是以脚本的方式执行任务的，需要完全吃透源码才可以调用，学习成本高，没有图形开发化界面和监控界面，运维成本相对高。 |
|               | 底层架构       | 分布式集群高可用架构，可以水平扩展到多节点支持超大数据量，架构容错性高，可以自动调节任务在节点之间分配，适用于大数据场景 | 主从结构非高可用，扩展性差，架构容错性低，不适用大数据场景   | 可做集群部署，规避单点故障，依赖于外部环境，如Oracle RAC等； | schema mapping非自动；可复制性比较差；更新换代不是很强       | 支持分布式部署                                               | 支持单机部署和集群部署两种方式                               |
| 功能          | CDC机制        | 基于日志、基于时间戳和自增序列等多种方式可选                 | 基于时间戳、触发器等                                         | 主要是基于日志                                               | 基于日志、基于时间戳和自增序列等多种方式可选                 | 基于触发器、基于时间戳和自增序列等多种方式可选               | 离线批处理                                                   |
|               | 对数据库的影响 | 基于日志的采集方式对数据库无侵入性                           | 对数据库表结构有要求，存在一定侵入性                         | 源端数据库需要预留额外的缓存空间                             | 基于日志的采集方式对数据库无侵入性                           | 有侵入性                                                     | 通过sql select 采集数据，对数据源没有侵入性                  |
|               | 自动断点续传   | 支持                                                         | 不支持                                                       | 支持                                                         | 不支持，依赖ETL设计的合理性（例如T-1），指定续读某个时间点的数据，非自动 | 不支持，依赖ETL设计的合理性（例如T-1），指定续读某个时间点的数据，非自动 | 不支持                                                       |
|               | 监控预警       | 可视化的过程监控，提供多样化的图表，辅助运维，故障问题可实时预警 | 依赖日志定位故障问题，往往只能是后处理的方式，缺少过程预警   | 无图形化的界面预警                                           | monitor可以看到报错信息，信息相对笼统，定位问题仍需依赖分析日志 | 有问题预警，定位问题仍需依赖日志                             | 依赖工具日志定位故障问题，没有图形化运维界面和预警机制，需要自定义开发。 |
|               | 数据清洗       | 围绕数据质量做轻量清洗                                       | 围绕数据仓库的数据需求进行建模计算，清洗功能相对复杂，需要手动编程 | 轻量清洗                                                     | 支持复杂逻辑的清洗和转化                                     | 支持复杂逻辑的清洗和转化                                     | 需要根据自身清晰规则编写清洗脚本，进行调用（DataX3.0 提供的功能）。 |
|               | 数据转换       | 自动化的schema mapping                                       | 手动配置schema mapping                                       | 需手动配置异构数据间的映射                                   | 手动配置schema mapping                                       | 手动配置schema mapping                                       | 通过编写json脚本进行schema mapping映射                       |
| 特性          | 数据实时性     | 实时                                                         | 非实时                                                       | 实时                                                         | 支持实时，但是主流应用都是基于时间戳等方式做批量处理，实时同步效率未知 | 实时                                                         | 定时                                                         |
|               | 应用难度       | 低                                                           | 高                                                           | 中                                                           | 高                                                           | 中                                                           | 高                                                           |
|               | 是否需要开发   | 否                                                           | 是                                                           | 是                                                           | 是                                                           | 是                                                           | 是                                                           |
|               | 易用性         | 高                                                           | 低                                                           | 中                                                           | 低                                                           | 低                                                           | 低                                                           |
|               | 稳定性         | 高                                                           | 低                                                           | 高                                                           | 中                                                           | 中                                                           | 中                                                           |
| 其他          | 实施及售后服务 | 原厂实施和售后服务                                           | 开源软件，需自客户自行实施、维护                             | 原厂和第三方的实施和售后服务                                 | 主要为第三方的实施和售后服务                                 | 分为开源版和企业版，企业版可提供相应服务                     | 阿里开源代码，需要客户自动实施、开发、维护                   |

## 八、DataX与Kettle对比

- [浅测评DataX与Kettle](https://blog.csdn.net/weixin_44411398/article/details/116695232)

- [Datax和Kettle使用场景的对比](https://blog.csdn.net/lzhcoder/article/details/120830522)

| 较维度\产品    | Kettle                                                       | DataX                                                        |                             |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | --------------------------- |
| 设计及架构     | 适用场景                                                     | 面向数据仓库建模传统ETL工具                                  | 面向数据仓库建模传统ETL工具 |
| 支持数据源     | 多数关系型数据库                                             | 少数关系型数据库和大数据非关系型数据库                       |                             |
| 开发语言       | Java                                                         | Python、Java                                                 |                             |
| 可视化web界面  | KettleOnline代码收费Kettle-manager代码免费                   | Data-Web代码免费                                             |                             |
| 底层架构       | 主从结构非高可用，扩展性差，架构容错性低，不适用大数据场景   | 支持单机部署和第三方调度的集群部署两种方式                   |                             |
| 功能           | CDC机制                                                      | 基于时间戳、触发器等                                         | 离线批处理                  |
| 抽取策略       | 支持增量，全量抽取                                           | 支持全量抽取。不支持增量抽取要通过shell脚本自己实现          |                             |
| 对数据库的影响 | 对数据库表结构有要求，存在一定侵入性                         | 通过sql select 采集数据，对数据源没有侵入性                  |                             |
| 自动断点续传   | 不支持                                                       | 不支持                                                       |                             |
| 数据清洗       | 围绕数据仓库的数据需求进行建模计算，清洗功能相对复杂，需要手动编程 | 需要根据自身清晰规则编写清洗脚本，进行调用（DataX3.0 提供的功能）。 |                             |
| 数据转换       | 手动配置schema mapping                                       | 通过编写json脚本进行schema mapping映射                       |                             |
| 特性           | 数据实时性                                                   | 非实时                                                       | 定时                        |
| 应用难度       | 高                                                           | 高                                                           |                             |
| 是否需要开发   | 是                                                           | 是                                                           |                             |
| 易用性         | 低                                                           | 低                                                           |                             |
| 稳定性         | 低                                                           | 中                                                           |                             |
| 抽取速度       | 小数据量的情况下差别不大，大数据量时datax比kettle快。datax对于数据库压力比较小 |                                                              |                             |
| 其他           | 实施及售后服务                                               | 开源软件，社区活跃度高                                       | 阿里开源代码，社区活跃度低  |

## 九、实战案例

- [dataX案例 读取mysql（通过表名或自定义sql语句）数据写入到hdfs中 (txt或orc存储，gzip或snappy压缩)_](https://blog.csdn.net/qq_41712271/article/details/108590074)
- [dataX案例 读取hdfs文件，写入到mysql中](https://blog.csdn.net/qq_41712271/article/details/108602602)
- [dataX案例 从Oracle中读取数据（自定义sql语句）存到MySQL中_](https://blog.csdn.net/qq_41712271/article/details/108604459)
- [dataX案例-从mysql读取数据，写入到hbase中](https://blog.csdn.net/qq_41712271/article/details/108610591)
- [dataX案例-从hbase中读取数据，写入到文本文件中](https://blog.csdn.net/qq_41712271/article/details/108618905)
- [使用 DataX 实现数据同步（高效的数据同步工具）](https://juejin.cn/post/7077744714954309669#heading-9)
- [DataX | MySQL多表全量同步和多表定时增量同步](https://blog.csdn.net/qq_25112523/article/details/109276879)

### 9.1 Datax同步MySQL到Hive

- [dataX同步mysql至hive](https://blog.csdn.net/AyubLIbra/article/details/115838882)

#### 9.1.1 前言

以下是我的一个mysql同步到Hive，相关的变量都可以通过传参统一脚本处理。

```json
{
    "job": {
        "setting": {
            "speed": {
                "channel": 3
            }
        },
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "${username}",
                        "password": "${password}",
                        "connection": [
                            {
                                "jdbcUrl": [
                                    "${jdbcUrl}"
                                ],
                                "querySql": [
                                    "select id,create_time,update_time from ${sourceTableName} where update_time<'${endTime}' "                                  
                                ]
                            }
                        ]
                    }
                },
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "column": [
                            {
                                "name": "id",
                                "type": "string"
                            },
                            {
                                "name": "create_time",
                                "type": "string"
                            },
                            {
                                "name": "update_time",
                                "type": "string"
                            }
                        ],
                        "isCompress": "${isCompress}",
                        "defaultFS": "${hdfsPath}",
                        "fieldDelimiter": "${fieldDelimiter}",
                        "fileName": "${fileName}",
                        "fileType": "${fileType}",
                        "path": "${path}",
                        "writeMode": "${writeMode}"
                    }
                }
            }
        ]
    }
}
```

#### 9.1.2 参数调用和传参

使用dataX调用这个脚本。

```shell
python ${DATAX_HOME}/bin/datax.py -p"-DtargetDBName=$TARGET_DB_NAME -DtargetTableName=$TARGET_TABLE_NAME  -DjdbcUrl=$MYSQL_URL -Dusername=$MYSQL_USERNAME -Dpassword=$MYSQL_PASSWD -DsourceTableName=$SOURCE_TABLE_NAME -DhdfsPath=$HDFS_PATH -DstartTime=${START_TIME} -DendTime=${END_TIME} -DisCompress=$ISCOMPRESS -DwriteMode=$WRITEMODE -DfileType=$FILETYPE -DfieldDelimiter='$FIELDDELIMITER' -DfileName=$TARGET_TABLE_NAME  -Dpath=${PATH_HIVE}$TARGET_DB_NAME.db/$TARGET_TABLE_NAME/day=$DT_TIME" $DATAX_JSON_FILE;
```

在这个命令中你会发现将所有的变量都通过shell命令传递进去了。后续的这些变量传递我在更新。之所以这么多变量其主要是为了方便后续的脚本更新和调度运行。

对于开发人员只需要关心主要逻辑就行了。

有了这个基础脚本，我们就可以将HIVE上的一些功能一起合并到shell脚本中：

- 增量同步，保留全部数据。
- 全量同步，全量同步只保留固定周期的历史全量。
- 刷新元数据。
- 通知更新成功。
- 多个mysql业务库的匹配。
- 生产业务库密码的保护。

#### 9.1.3 封装shell调用脚本

基于上面的考虑。封装dataX的调用脚本。

```shell
#!/bin/bash
source /etc/profile
DATAX_HOME="/home/data/datax"
SHELL_PATH="/home/data/dw_datax"
SCRIPT_PATH=${SHELL_PATH}/job
DATAX_LOG=${SHELL_PATH}/logs/datax.log.`date "+%Y-%m-%d"`
HDFS_PATH="hdfs://hdfs-cluster"
#START_TIME=$(date -d "-1 day" +%Y-%m-%d)
#END_TIME=$(date "+%Y-%m-%d")
#DT_TIME=$(date -d "-1 day" +%Y%m%d)
START_TIME=""
END_TIME=""
DT_TIME=""
#失效日期
INVALID_DATE=""
#失效天数
INVALID_DAYS=180
#是否清除失效数据：默认清除
IS_CLEAR_INVALID_DATA=1

#参数
ISCOMPRESS="false"
WRITEMODE="nonConflict"
FIELDDELIMITER="|"
FILETYPE="orc"
PATH_HIVE="/user/hive/warehouse/"
MYSQL_URL=""
#数据库用户名
MYSQL_USERNAME="admin"
#数据库密码
MYSQL_PASSWD="123456"
#默认同步目标库名
TARGET_DB_NAME="ods"
#同步源库名
SOURCE_DB_NAME=""
#同步源表名
SOURCE_TABLE_NAME=""
#业务名称
BUSINESS_NAME=""
#datax json文件
DATAX_JSON_FILE=/temp


# 数据库实例信息
declare -A db_instance_conf
# 数据库用户名
declare -A db_instance_user_conf
# 数据库密码
declare -A db_instance_pwd_conf
# 数据库实例与库映射关系
declare -A db_instance_maps

# 初始化数据库实例配置
function initInstanceConf()
{
        # 主业务线 ywx1
        db_instance_conf["db_main_data"]="jdbc:mysql://192.168.1.1:3306/"
        db_instance_user_conf["db_main_data"]="admin"
        db_instance_pwd_conf["db_main_data"]="123456"
         # 业务线2 ywx2
        db_instance_conf["db_data"]="jdbc:mysql://192.168.1.2:3306/"
        db_instance_user_conf["db_data"]="admin"
        db_instance_pwd_conf["db_data"]="123456"
        ...
        
}

# 初始化库和数据库实例映射关系
function initDbAndInstanceMaps()
{
        #主业务线
        db_instance_maps["ywx1_db_main"]="db_main_data"
        
        #业务线2
        db_instance_maps["ywx2_db_data"]="db_data"
        #业务线3
        db_instance_maps["ywx3_db_insurance"]="db_ywx3"
        
        ...
        
        ...
        
        db_instance_maps["dss_db_dss"]="db_dss"
		
}

#时间处理  传入参数 yyyy-mm-dd
function DateProcess()
{
echo "日期时间为"$1
if echo $1 | grep -Eq "[0-9]{4}-[0-9]{2}-[0-9]{2}" && date -d $1 +%Y%m%d > /dev/null 2>&1
  then :
        START_TIME=$(date -d $1 "+%Y-%m-%d")
        END_TIME=$(date -d "$1 +1 day" +%Y-%m-%d)
        DT_TIME=$(date -d $1 +"%Y%m%d")
        INVALID_DATE=$(date -d "$1 -$INVALID_DAYS day" +%Y%m%d)
        echo 时间正确: $START_TIME / $END_TIME / $DT_TIME / $INVALID_DATE;
else
  echo "输入的日期格式不正确，应为yyyy-mm-dd";
  exit 1;
fi;

}

function DataConnect()
{
        db_business_key="$BUSINESS_NAME""_""$SOURCE_DB_NAME"
        db_instance_key=${db_instance_maps["$db_business_key"]}
        echo $db_business_key $db_instance_key
        if [ ! -n "$db_instance_key" ]; then
                echo "当前数据库连接信息不存在，请确认业务和数据库连接是否正确或联系管理员添加"
                exit 1;
        fi
        db_instance_value=${db_instance_conf["$db_instance_key"]}
	MYSQL_USERNAME=${db_instance_user_conf["$db_instance_key"]}
        MYSQL_PASSWD=${db_instance_pwd_conf["$db_instance_key"]}
	echo $db_instance_value
        if [ ! -n "$db_instance_value" ]; then
                echo "当前数据库连接信息不存在，请确认业务和数据库连接是否正确或联系管理员添加"
                exit 1;
        fi
        MYSQL_URL="$db_instance_value$SOURCE_DB_NAME"
}

#每天运行 执行dataX
function BaseDataxMysql2Hive()
{
        #清除重复同步数据分区&新增分区
        hive -e "ALTER TABLE $TARGET_DB_NAME.$TARGET_TABLE_NAME DROP IF EXISTS PARTITION(day='$DT_TIME');ALTER TABLE $TARGET_DB_NAME.$TARGET_TABLE_NAME ADD IF NOT EXISTS PARTITION (day='$DT_TIME')";
        #执行同步
        echo "开始执行同步"
    if ! python ${DATAX_HOME}/bin/datax.py -p"-DtargetDBName=$TARGET_DB_NAME -DtargetTableName=$TARGET_TABLE_NAME  -DjdbcUrl=$MYSQL_URL -Dusername=$MYSQL_USERNAME -Dpassword=$MYSQL_PASSWD -DsourceTableName=$SOURCE_TABLE_NAME -DhdfsPath=$HDFS_PATH -DstartTime=${START_TIME} -DendTime=${END_TIME} -DisCompress=$ISCOMPRESS -DwriteMode=$WRITEMODE -DfileType=$FILETYPE -DfieldDelimiter='$FIELDDELIMITER' -DfileName=$TARGET_TABLE_NAME  -Dpath=${PATH_HIVE}$TARGET_DB_NAME.db/$TARGET_TABLE_NAME/pt_day=$DT_TIME" $DATAX_JSON_FILE;then
        echo "command failed"
        exit 1;
        fi

    echo "同步结束"
    #删除定义的失效日期数据
    if(($IS_CLEAR_INVALID_DATA==1));then
        echo "清除失效$INVALID_DATE天数的历史数据"
        hive -e "ALTER TABLE $TARGET_DB_NAME.$TARGET_TABLE_NAME DROP IF EXISTS PARTITION (pt_day<=${INVALID_DATE});"
        fi
    #同步分区元数据
        #hive -e "ANALYZE TABLE $TARGET_DB_NAME.$TARGET_TABLE_NAME PARTITION (day=${DT_TIME}) COMPUTE STATISTICS;"
        #删除分区数据
}

function parseArgs()
{
        while getopts ":d:ab:s:m:f:t:n:u:p:" opt
        do
            case $opt in
                d)
            echo "参数d的值$OPTARG"
                DateProcess $OPTARG
                ;;
                a)
                IS_CLEAR_INVALID_DATA=0
                echo "参数a的值$OPTARG"
                ;;
                b)
                echo "参数b的值$OPTARG"
                BUSINESS_NAME=$OPTARG
                ;;
                m)
                echo "参数m的值$OPTARG"
                SOURCE_DB_NAME=$OPTARG
                ;;
                s)
                echo "参数s的值$OPTARG"
                SOURCE_TABLE_NAME=$OPTARG
                ;;
                f)
                echo "参数f的值$OPTARG"
                DATAX_JSON_FILE=$OPTARG
                ;;
                n)
                echo "参数n的值$OPTARG"
                TARGET_DB_NAME=$OPTARG
                ;;
                t)
                echo "参数t的值$OPTARG"
                TARGET_TABLE_NAME=$OPTARG
                ;;
                u)
                echo "参数u的值$OPTARG"
                MYSQL_USERNAME=$OPTARG
                ;;
                p)
                echo "参数t的值$OPTARG"
                MYSQL_PASSWD=$OPTARG
                ;;
                ?)
                echo "未知参数"
                exit 1;;
                :)
            echo "没有输入任何选项 $OPTARG"
            ;;
        esac done
}

function judgeParams()
{
        if  [ ! -n "$DT_TIME" ] ;then
            echo "you have not input a etlDate! format {-d yyyy-mm-dd} "
            exit 1;
        fi

        if  [ ! -n "$BUSINESS_NAME" ] ;then
            echo "you have not input a businessName! incloud(xxx,xxxx,x,xx) example {-b xxx}"
            exit 1;
        fi

        if  [ ! -n "$SOURCE_DB_NAME" ] ;then
            echo "you have not input a sourceDB!"
            exit 1;
        fi

        if  [ ! -n "$SOURCE_TABLE_NAME" ] ;then
            echo "you have not input a sourceTable example {-s user_info}!"
            exit 1;
        fi

        if  [ ! -n "$DATAX_JSON_FILE" ] ;then
            echo "you have not input a dataxJson! example {-f ods_ywx1_user_info_di.json}"
            exit 1;
        fi
        if  [ ! -n "$TARGET_TABLE_NAME" ] ;then
            echo "you have not input a targetTable! example {-t ods_ywx1_user_info_di}"
            exit 1;
        fi
}


function startsync()
{
        #初始化数据库实例
        initInstanceConf
        #初始化库和数据库实例映射关系
        initDbAndInstanceMaps
        #解析参数
        parseArgs "$@"
        #初始化数据链接
        DataConnect
        #判断参数
        judgeParams
        #同步数据
        BaseDataxMysql2Hive
}

# -d: 处理时间
# -b：业务线 (ywx,ywx1,ywx1,...,ywxn)
# -m：源数据库
# -a：增量数据不清除分区数据：默认清除
# -s：源数据表
# -n：目标数据库
# -t：目标数据表
# -f：datax同步json文件
# -p：密码
# -u：用户名

startsync "$@"
```

有了这个shell脚本，后续对于同步的一些同步完成功能的通知以及新功能都可以新增。同时又新形成了一个数据同步的规范性和开发的规范性。

#### 9.1.4 调度平台调度脚本

有了上面的脚本，我们就可以只需要写好源表和目的表的名称。同时通过 -a 来区别增量还是全量同步进行处理。

```bash
#源表 -s
SOURCE_TABLE_NAME="user_info"
#目标表 -t
TARGET_TABLE_NAME="ods_main_user_info_df"
#datax 文件 -f
DATAX_FILE="${BASE_DIR_PATH}/ods_main_user_info_df.json"
ETL_DATE=${ETL_DATE}
BUSINESS_NAME=${BUSINESS_NAME}
SOURCE_DB_NAME=${SOURCE_DB_NAME}
#!/bin/bash
source /etc/profile
sh dataxsync.sh -d $ETL_DATE -b $BUSINESS_NAME -m $SOURCE_DB_NAME -s $SOURCE_TABLE_NAME -t $TARGET_TABLE_NAME -f $DATAX_FILE
```

我们可以发现这个脚本中包含了四个变量：

```bash
${BASE_DIR_PATH}
${ETL_DATE}
${BUSINESS_NAME}
${SOURCE_DB_NAME}
```

这几个变量主要是通过调度平台传入，

- **BASE_DIR_PATH**：dataX脚本的统一地址，之所以弄这个目录主要是为了区分不同业务线。

- **ETL_DATE**：每天同步的时间 ： yyyy-mm-dd，同时我们可以再脚本上多增加几个时间，通过这个变量转换出来（yyyyMMdd, yyyyMMdd-1…）

- **BUSINESS_NAME**：业务线的标识，我们也是可以用主题域区分。主要是用来识别数据库

- **SOURCE_DB_NAME**：业务库的表名。

### 9.2 MySQL到MySQL

```json
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "123123",
                        "column": [ "*" ],
                        "splitPk": "ID",
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:mysql://192.168.1.1:3306/course-study?useUnicode=true&characterEncoding=utf8"],
                                "querySql": [
                                    "select db_id,on_line_flag from db_info where db_id < 10;"
                                ],
                            }
                        ]
                    }
                },
                "writer": {
                    "name": "mysqlwriter",
                    "parameter": {
                        "column": ["*"],
                        "connection": [
                            {"jdbcUrl": "jdbc:mysql://192.168.1.2:3306/course-study?useUnicode=true&characterEncoding=utf8",
                             "table": ["t_member"]
                            }
                        ],
                        "password": "123123",
                        "preSql": ["truncate t_member"],
                        "session": ["set session sql_mode='ANSI'"],
                        "username": "root",
                        "writeMode": "insert"
                    }
                }
            }
        ],
        "setting": {"speed": {"channel": "5"}}
    }
}
```

### 9.3 DataX支持MySQL-8读写

- [DataX支持MySQL-8读写](https://blog.csdn.net/docsz/article/details/124041439)

#### 9.3.1 mysqlwrite模块

**1.1、修改pom**

如果只用到了mysql的写，可以只修改mysqlwriter目录下的pom文件

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <!--            <version>${mysql.driver.version}</version>-->
    <version>8.0.28</version>
</dependency>
```

#### 9.3.2 修改DataBaseType

**2.1、打开DataBaseType类**

```powershell
vi plugin-rdbms-util/src/main/java/com/alibaba/datax/plugin/rdbms/util/DataBaseType.java
```

**2.2、全文替换**

```powershell
:%s/convertToNull/CONVERT_TO_NULL/
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/a1b166689ae24026b635e4b68176e63a.png)

**2.3、修改jdbc驱动的名称**

**com.mysql.jdbc.Driver 为 com.mysql.cj.jdbc.Driver**

plugin-rdbms-util/src/main/java/com/alibaba/[datax](https://so.csdn.net/so/search?q=datax&spm=1001.2101.3001.7020)/plugin/rdbms/util/DataBaseType.java

![在这里插入图片描述](https://img-blog.csdnimg.cn/9853c4ddda664c66a2949663965a532b.png)

adswriter/src/main/java/com/alibaba/datax/plugin/writer/adswriter/load/AdsHelper.java

![在这里插入图片描述](https://img-blog.csdnimg.cn/8fc4ff58054a43e7ad351a1182ea5365.png)

#### 9.3.3 编译打包

打包成功后，查看datax/plugin/writer/mysqlwriter/libs/，已经包含新版的驱动jar包

```powershell
mvn -U clean package assembly:assembly -Dmaven.test.skip=true
```

#### 9.3.4 mysql-jdbc连接

```powershell
"jdbcUrl":"jdbc:mysql://ip:port/test?useUnicode=true&characterEncoding=UTF-8&useSSL=false",
```

## 附录

### A.1 DataX系列博文

掘金：政采云团队

- [DataX 全系列之一 —— DataX 安装和使用](https://juejin.cn/post/7006617025816559653)
- [DataX全系列之二 —— DataX 总体架构和原理](https://juejin.cn/post/7006658351375335431)
- [DataX 全系列之三 —— DataX 源码运行流程分析](https://juejin.cn/post/7006619232641220616)
- [DataX 全系列之四 —— DataX 核心数据结构](https://juejin.cn/post/7007616850745884708)
- [DataX 全系列之五 —— DataX-web 介绍和使用](https://juejin.cn/post/7006658574529069086)

阿里云社区：阿甘兄

- [DataX教程（01）- 入门](https://developer.aliyun.com/article/1045779)
- [DataX教程（02）- IDEA运行DataX完整流程（填完所有的坑）](https://developer.aliyun.com/article/1045790)
- [DataX教程（03）- 源码解读（超详细版）](https://developer.aliyun.com/article/1045805)
- [DataX教程（04）- 配置完整解读](https://developer.aliyun.com/article/1045834)
- [DataX教程（05）- DataX Web项目实践](https://developer.aliyun.com/article/1045858)
- [DataX教程（07）- 图解DataX任务分配及执行流程](https://developer.aliyun.com/article/1045866)
- [DataX教程（08）- 监控与汇报](https://developer.aliyun.com/article/1045885)
- [DataX教程（09）- DataX是如何做到限速的？](https://developer.aliyun.com/article/1045898)
- [DataX教程（10）- DataX插件热插拔原理](https://developer.aliyun.com/article/1045912)
