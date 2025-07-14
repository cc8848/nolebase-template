- [数据治理之元数据管理的利器——Atlas](https://www.cnblogs.com/tree1123/p/15480639.html)

## 一、数据治理与元数据管理

### 1.1 背景

为什么要做数据治理？ 业务繁多，数据繁多，业务数据不断迭代。人员流动，文档不全，逻辑不清楚，对于数据很难直观理解，后期很难维护。

在大数据研发中，原始数据就有着非常多的数据库，数据表。

而经过数据的聚合以后，又会有很多的维度表。

近几年来数据的量级在疯狂的增长，由此带来了系列的问题。作为对人工智能团队的数据支撑，我们听到的最多的质疑是 “正确的数据集”，他们需要正确的数据用于他们的分析。我们开始意识到，虽然我们构建了高度可扩展的数据存储，实时计算等等能力，但是我们的团队仍然在浪费时间寻找合适的数据集来进行分析。

也就是我们缺乏对数据资产的管理。事实上，有很多公司都提供了开源的解决方案来解决上述问题，这也就是数据发现与元数据管理工具。

### 1.2 元数据管理

**什么是元数据：**

举几个例子：

- 如果一本书是一个“数据"，那么它的书名、封面、出版社、作者、总页码就是它的“元数据”。
- 如果数据库中某个表是一个”数据”，那么它的列名、列类型、列长度、表注释就是它的"元数据"。

元数据用来描述数据的数据，通过描述数据的产生、存储、使用情况、业务含义等信息，以及数据管理人员相关信息。让人们能够清楚拥有什么数据、代表什么、源自何处、如何在系统中移动，以及哪些人可以使用源数据，如何使用。

简单地说，元数据管理是为了对数据资产进行有效的组织。它使用元数据来帮助管理他们的数据。它还可以帮助数据专业人员收集、组织、访问和丰富元数据，以支持数据治理。

三十年前，数据资产可能是 Oracle 数据库中的一张表。然而，在现代企业中，我们拥有一系列令人眼花缭乱的不同类型的数据资产。可能是关系数据库或 NoSQL 存储中的表、实时流数据、 AI 系统中的功能、指标平台中的指标，数据可视化工具中的仪表板。

现代元数据管理应包含所有这些类型的数据资产，并使数据工作者能够更高效地使用这些资产完成工作。

所以，元数据管理应具备的功能如下：

- **搜索和发现：**数据表、字段、标签、使用信息
- **访问控制：**访问控制组、用户、策略
- **数据血缘：**管道执行、查询
- **合规性：**数据隐私/合规性注释类型的分类
- **数据管理：**数据源配置、摄取配置、保留配置、数据清除策略
- **AI 可解释性、再现性：**特征定义、模型定义、训练运行执行、问题陈述
- **数据操作：**管道执行、处理的数据分区、数据统计
- **数据质量：**数据质量规则定义、规则执行结果、数据统计

### 1.3 架构与开源方案

下面介绍元数据管理的架构实现，不同的架构都对应了不同的开源实现。

下图描述了第一代元数据架构。它通常是一个经典的单体前端（可能是一个 Flask 应用程序），连接到主要存储进行查询（通常是 MySQL/Postgres），一个用于提供搜索查询的搜索索引（通常是 Elasticsearch），并且对于这种架构的第 1.5 代，也许一旦达到关系数据库的“递归查询”限制，就使用了处理谱系（通常是 Neo4j）图形查询的图形索引。

![img](https://img-blog.csdnimg.cn/img_convert/9570647e2edaf7123370eeffa9d30597.jpeg)

很快，第二代的架构出现了。单体应用程序已拆分为位于元数据存储数据库前面的服务。该服务提供了一个 API，允许使用推送机制将元数据写入系统。

![img](https://img-blog.csdnimg.cn/img_convert/8eb4b5f20c73926ff4293b7e8fb3c08c.jpeg)

第三代架构是基于事件的元数据管理架构，客户可以根据他们的需要以不同的方式与元数据数据库交互。

元数据的低延迟查找、对元数据属性进行全文和排名搜索的能力、对元数据关系的图形查询以及全扫描和分析能力。

![img](https://img-blog.csdnimg.cn/img_convert/f8b7a36406012f0ae242331877e5313f.jpeg)

Apache Atlas 就是采用的这种架构，并且与Hadoop 生态系统紧密耦合。下图是当今元数据格局的简单直观表示：（包含部分非开源方案）

![img](https://img-blog.csdnimg.cn/img_convert/eb142d2f2c39982cba037ffd2944d9c8.jpeg)

## 二、Apache Atlas简介

### 2.1 简介

在当今大数据的应用越来越广泛的情况下，数据治理一直是企业面临的巨大问题。

大部分公司只是单纯的对数据进行了处理，而数据的血缘，分类等等却很难实现，市场上也急需要一个专注于数据治理的技术框架，这时Atlas应运而生。

Atlas官网地址：https://atlas.apache.org/，Atlas是Hadoop的数据治理和元数据框架。

> 文档查看地址：https://atlas.apache.org/2.1.0/index.html
>
> 下载地址：https://www.apache.org/dyn/closer.[cgi](https://so.csdn.net/so/search?q=cgi&spm=1001.2101.3001.7020)/atlas/2.1.0/apache-atlas-2.1.0-sources.tar.gz

Atlas是一组可扩展和可扩展的核心基础治理服务，使企业能够有效，高效地满足Hadoop中的合规性要求，并允许与整个企业数据生态系统集成。

Apache Atlas为组织提供了开放的元数据管理和治理功能，以建立其数据资产的目录，对这些资产进行分类和治理，并为数据科学家，分析师和数据治理团队提供围绕这些数据资产的协作功能。

如果想要对这些数据做好管理，光用文字、文档等东西是不够的，必须用图。Atlas就是把元数据变成图的工具。

![img](https://img-blog.csdnimg.cn/img_convert/109c1666f77dd236dd8636e1b6a96dff.png)

### 2.2 特性

- Atlas支持各种Hadoop和非Hadoop元数据类型
- 提供了丰富的REST API进行集成
- 对数据血缘的追溯达到了字段级别，这种技术还没有其实类似框架可以实现
- 对权限也有很好的控制

Atlas包括以下组件：

- 采用Hbase存储元数据
- 采用Solr实现索引
- Ingest/Export 采集导出组件 Type System类型系统 Graph Engine图形引擎 共同构成Atlas的核心机制
- 所有功能通过API向用户提供，也可以通过Kafka消息系统进行集成
- Atlas支持各种源获取元数据：Hive，Sqoop，Storm。。。
- 还有优秀的UI支持

Apache Atlas为Hadoop的元数据治理提供了以下特性：

**数据分类**

- 为元数据导入或定义业务导向的分类注释

- 定义，注释，以及自动捕获数据集和底层元素之间的关系

- 导出元数据到第三方系统

**集中审计**

- 捕获与所有应用，过程以及与数据交互的安全访问信息

- 捕获执行，步骤，活动等操作的信息

**搜索与血缘**

- 预定义的导航路径用来探索数据分类以及审计信息

- 基于文本的搜索特性来快速和准确的定位相关联的数据和审计事件

- 对数据集血缘关系的可视化浏览使用户可以下钻到操作，安全以及数据起源相关的信息

**安全与策略引擎**

- 基于数据分类模式，属性以及角色的运行时合理合规策略

- 基于分类-预测的高级策略定义以防止数据推导

- 基于cell的属性和值的行/列级别的masking

### 2.3 Atlas的主要功能

`元数据管理和治理`，提升团队间对于数据资产的协作功能

| 具体功能   | 详细介绍                                                     |
| ---------- | ------------------------------------------------------------ |
| 元数据分类 | 支持对元数据进行`分类管理`，例如个人信息，敏感信息等         |
| 元数据检索 | 可按照元数据类型、元数据分类进行`检索`，支持全文检索         |
| 元数据血缘 | 支持`表到表`和`字段到字段`之间的`血缘依赖`，便于进行`问题回溯和影响分析`等 |

### 2.4 整体流程

元数据处理的整体流程入下图所示：

![img](https://img-blog.csdnimg.cn/20200726150857312.png)

在Atlas中查询某一个元数据对象时往往需要遍历图数据库中的多个顶点与边，相比关系型数据库直接查询一行数据要复杂的多，当然使用图数据库作为底层存储也存在它的优势，比如可以支持复杂的数据类型和更好的支持血缘数据的读写。

**Atlas获取Hive元数据的原理：**

- **首先执行import-hive.sh，这里相当于初始化作用，Altas获取Hive的库/表结构（注意！此时并没有获获表与表/字段之间的血缘关系，只是获取了表的结构，即 create table语句，此时可以在web ui上面只会看到单个表的名字，location等等信息）**

- **要想正真获取血缘关系，必须配置`hive.exec.post.hooks`，然后需要把任务重新调度一下，即执行 `insert overwrite a select \* from b`(注意！a表中一定要有新的数据落地)。此时配置的hook 会监听感知到hive表中有更新操作，然后通过Kafka将更新的数据发给Atlas，Atlas会对数据修改，这样在Web Ui 就会看到 a表与b表的血缘关系。**

- Atlas的缺点：

  在实际使用中，发现Atlas存在的一些缺点

  - 1.两个表之间必须要有真实的数据落地，才能解析出来表的血缘关系。例如：查询一条在a表中不存在的数据，然后插入b表。这样就无法解析ab表的血缘关系。
  - 2.如果sql中用到的表，使用 `with as...`当做临时表。这样Atlas也无法解析血缘关系。

### 2.5 类型系统

Atlas 允许用户为他们想要管理的元数据对象定义一个模型。该模型由称为 “类型” (type)的定义组成。被称为 “实体” (entities)的 “类型” 实例表示被管理的实际元数据对象。由 Atlas 管理的所有元数据对象（例如Hive表）都使用类型进行建模，并表示为实体。

- Type：**Atlas中的 “类型” 定义了如何存储和访问特定类型的元数据对象**。类型表示了所定义元数据对象的一个或多个属性集合。具有开发背景的用户可以将 “类型” 理解成面向对象的编程语言的 “类” 定义的或关系数据库的 “表模式”。类型具有元类型，元类型表示 Atlas 中此模型的类型：

  - 基本元类型： Int，String，Boolean等
    - 集合元类型：例如Array，Map

  - Class，Struct，Trait

- Entities：Atlas中的 **“实体” 是类 “类型” 的特定值或实例**，因此表示真实世界中的特定元数据对象。回顾我们的面向对象编程语言的类比，“实例” 是某个 “类” 的 “对象”。

- Attributes：Atlas中的属性还有一些属性，其定义了与类型系统相关的更多概念，包括：

  - isComposite - 是否复合

  - isIndexable - 是否索引

  - isUnique - 是否唯一

  - multiplicity - 指示此属性是（必需的／可选的／还是可以是多值）的

Atlas 提供了一些预定义的系统类型：

- Referenceable：此类型表示可使用名为 qualifiedName 的唯一属性搜索的所有实体

- Asset：此类型包含名称，说明和所有者等属性

- Infrastructure：此类型扩展了Referenceable和Asset ，通常可用于基础设施元数据对象（如群集，主机等）的常用超类型

- DataSet：此类型扩展了Referenceable和Asset 。在概念上，它可以用于表示存储数据的类型。在 Atlas 中，hive表，Sqoop RDBMS表等都是从 DataSet 扩展的类型。扩展 DataSet 的类型可以期望具有模式，它们将具有定义该数据集的属性的属性。例如， hive_table 中的 columns 属性。另外，扩展 DataSet 的实体类型的实体参与数据转换，这种转换可以由 Atlas 通过 lineage（或 provenance）生成图形。

- Process：此类型扩展了Referenceable和Asset 。在概念上，它可以用于表示任何数据变换操作。例如，将原始数据的 hive 表转换为存储某个聚合的另一个 hive 表的 ETL 过程可以是扩展过程类型的特定类型。流程类型有两个特定的属性，输入和输出。

Hive表是Atlas本机定义的一种类型的示例，定义Hive表具有以下属性：

```bash
Name:         hive_table
TypeCategory: Entity
SuperTypes:   DataSet
Attributes:    
	name:              string    
	db:                 hive_db    
	owner:             string    
	createTime:       date    
	lastAccessTime:   date    
	comment:           string    
	retention:         int    
	sd:                 hive_storagedesc    
	partitionKeys:    array<hive_column>    
	aliases:           array<string>    
	columns:           array<hive_column>    
	parameters:        map<string>    
	viewOriginalText: string    
	viewExpandedText: string    
	tableType:         string    
	temporary:         boolean
```

### 2.6 效果图

![img](https://img-blog.csdnimg.cn/img_convert/57031c2061d2d1f9e97a8fa03c58c402.png)

![img](https://img-blog.csdnimg.cn/img_convert/0ee87dd078b74926ced6c7d4de158da8.png)

## 三、整体架构

### 3.1 架构图

Atlas架构图如下：

![img](https://img-blog.csdnimg.cn/img_convert/02542865bad00635a8b80fa5628d09d8.png)

### 3.2 Core层

Atlas核心包含以下组件：

**类型(Type)系统**: Atlas允许用户为他们想要管理的元数据对象定义模型。该模型由称为“类型”的定义组成。称为“实体”的“类型”实例表示受管理的实际元数据对象。 Type System是一个允许用户定义和管理类型和实体的组件。开箱即用的Atlas管理的所有元数据对象（例如Hive表）都使用类型建模并表示为实体。要在Atlas中存储新类型的元数据，需要了解类型系统组件的概念。

需要注意的一个关键点是Atlas中建模的一般特性允许数据管理员和集成商定义技术元数据和业务元数据。也可以使用Atlas的功能定义两者之间的丰富关系。

**图形引擎**: Atlas在内部使用Graph模型持久保存它管理的元数据对象。这种方法提供了很大的灵活性，可以有效地处理元数据对象之间的丰富关系。图形引擎组件负责在Atlas类型系统的类型和实体之间进行转换，以及底层图形持久性模型。除了管理图形对象之外，图形引擎还为元数据对象创建适当的索引，以便可以有效地搜索它们。 Atlas使用JanusGraph存储元数据对象。

**采集/导出**：采集组件允许将元数据添加到Atlas。同样，“导出”组件将Atlas检测到的元数据更改公开为事件。消费者可以使用这些更改事件来实时响应元数据的变更。

### 3.3 Integration层

在Atlas中，用户可以使用以下的两种方式管理元数据：

**API**: Atlas的所有功能都通过REST API向最终用户暴露，该API允许创建，更新和删除类型和实体。它也是查询和发现Atlas管理的类型和实体的主要机制。

**Messaging**: 除了API之外，用户还可以选择使用基于Kafka的消息传递接口与Atlas集成。这对于将元数据对象传递到Atlas以及使用Atlas使用可以构建应用程序的元数据更改事件都很有用。如果希望使用与Atlas更松散耦合的集成来实现更好的可伸缩性，可靠性等，则消息传递接口特别有用.Atlas使用Apache Kafka作为通知服务器，用于钩子和元数据通知事件的下游消费者之间的通信。事件由钩子和Atlas写入不同的Kafka主题。

### 3.4 Metadata sources层

Atlas支持开箱即用的多种元数据源集成。未来还将增加更多集成。目前，Atlas支持从以下来源提取和管理元数据：

- Hive：通过hive bridge， atlas可以接入Hive的元数据，包括hive_db/hive_table/hive_column/hive_process
- Sqoop：通过sqoop bridge，atlas可以接入关系型数据库的元数据，包括sqoop_operation_type/ sqoop_dbstore_usage/sqoop_process/sqoop_dbdatastore
- Falcon：通过falcon bridge，atlas可以接入Falcon的元数据，包括falcon_cluster/falcon_feed/falcon_feed_creation/falcon_feed_replication/ falcon_process
- Storm：通过storm bridge，atlas可以接入流式处理的元数据，包括storm_topology/storm_spout/storm_bolt
- HBase: 通过hbase bridge

Atlas集成大数据组件的元数据源需要实现以下两点：

- 首先，需要基于atlas的类型系统定义能够表达大数据组件元数据对象的元数据模型(例如Hive的元数据模型实现在org.apache.atlas.hive.model.HiveDataModelGenerator)；

- 然后，需要提供hook组件去从大数据组件的元数据源中提取元数据对象，实时侦听元数据的变更并反馈给atlas；

集成意味着两件事：Atlas定义的元数据模型用于表示这些组件的对象。 Atlas提供了从这些组件中摄取元数据对象的组件（在某些情况下实时或以批处理模式）。

### 3.5 Applications层

Atlas管理的元数据被各种应用程序使用，以满足许多治理需求。

**Atlas Admin UI**: 该组件是一个基于Web的应用程序，允许数据管理员和科学家发现和注释元数据。这里最重要的是搜索界面和类似SQL的查询语言，可用于查询Atlas管理的元数据类型和对象。 Admin UI使用Atlas的REST API来构建其功能。

**Tag Based Policies**:Apache Ranger是Hadoop生态系统的高级安全管理解决方案，可与各种Hadoop组件进行广泛集成。通过与Atlas集成，Ranger允许安全管理员定义元数据驱动的安全策略以实现有效的治理。 Ranger是Atlas通知的元数据更改事件的使用者。

Business Taxonomy：从元数据源获取到 Atlas 的元数据对象主要是一种技术形式的元数据。为了增强可发现性和治理能力，Atlas 提供了一个业务分类界面，允许用户首先定义一组代表其业务域的业务术语，并将其与 Atlas 管理的元数据实体相关联。业务分类法是一种 Web 应用程序，目前是 Atlas Admin UI 的一部分，并且使用 REST API 与 Atlas 集成。

## 四、编译、部署与安装

Atlas的安装坑较多。本教程将详细介绍Atlas2.1.0整个安装过程，比较难受的是 ，Atlas不提供安装包，下载的是源码包 ，需要自行编译打包。

- [Atlas2.1.0实战：安装、配置、导入hive元数据、编译排坑](https://blog.csdn.net/qq_32656561/article/details/122456098)

### 4.1 下载

请前往官网 https://atlas.apache.org/#/Downloads

![img](https://img-blog.csdnimg.cn/img_convert/a5398e156b8c5c45a60dcd27b63aea60.png)

下载对应版本的源码包 本文使用的是 2.1.0版本

国内站点 速度要快一些

**https://mirrors.tuna.tsinghua.edu.cn/apache/atlas/2.1.0/apache-atlas-2.1.0-sources.tar.gz**

### 4.2 安装maven

注意，需要先安装maven，因为这是使用maven开发的java web工程。maven3.6.3版本即可

一些下载地址

http://maven.apache.org/download.cgi

https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

**上传到linux的/usr/local目录**

```bash
cd /usr/local
```

**解压文件**

```python
tar -zxvf apache-maven-3.6.3-bin.tar.gz
```

**配置环境变量**

```bash
vi /etc/profile
export MAVEN_HOME=/usr/local/apache-maven-3.6.3
export PATH=MAVEN_HOME/bin:PATH 
```

**刷新环境变量**

```bash
source /etc/profile
```

**检查版本**

```armasm
mvn -v 
```

![img](https://img-blog.csdnimg.cn/img_convert/466c649c2117e4a97cd79e38b4d2d058.png)

配置好maven的环境变量
注意国内需要配置好maven的国内镜像，一般使用阿里，华为，清华等镜像
vi /usr/local/apache-maven-3.6.3/conf/settings.xml

```xml
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name> <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
```

把这个setting.xml 拷贝到 ~/.m2/

cp settings.xml .m2/

### 4.3 编译Atlas

解压源码包

```undefined
tar -zxvf apache-atlas-2.1.0-sources.tar.gz 
```

##### 1.修改atlas源码工程的pom.xml

将hbase zookeeper hive等依赖的版本修改成自己环境中一致的版本（或者兼容的版本）

```xml
父工程pom文件
<zookeeper.version>3.4.14</zookeeper.version>
<hbase.version>2.2.3</hbase.version>
<solr.version>7.7.2</solr.version>
```

##### 2.执行maven编译打包

atlas可以使用内嵌的hbase-solr作为底层索引存储和搜索组件，也可以使用外置的hbase和solr
如果要使用内嵌的hbase-solr，则使用如下命令进行编译打包
cd apache-atlas-sources-2.1.0/
export MAVEN_OPTS="-Xms2g -Xmx2g"
mvn clean -DskipTests package -Pdist,embedded-hbase-solr

不用内置就这样 mvn clean -DskipTests package -Pdist

改完路径还会非常快的 耐心等待

![img](https://img-blog.csdnimg.cn/img_convert/6f96c4c42d7febe51414e53c408a7f6f.png)

atlas的webui子模块中依赖了nodejs，会从nodejs的中央仓库去下载相关依赖库

编译完成之后，会产生打包结果，所在位置是：源码目录中的新出现的distro/target目录

- 注意，这里产出的有压缩包也有解压后的包。这里面的hook包，可以看到有各种平台的hook包。
- 顾名思义，这就是钩子包，也就是各个大数据框架会提供各种生命周期回调函数，并且将相关信息以数据方式提供出来。这些钩子就可以去监听和接收数据

如果没有错误 并看到一排success就是成功了

![img](https://img-blog.csdnimg.cn/img_convert/d9ae2cc55ce0edb0b25f82dfe864c644.png)

### 4.4 安装Atlas

在完成Atlas编译以后，就可以进行Atlas的安装了。Atlas的安装主要是安装Atlas的Server端，也就Atlas的管理页面，并确保Atlas与Kafka Hbase Solr等组件的集成。

Atlas的系统架构如下，在确保 底层存储与UI界面正常后，之后就可以进行与Hive等组件的集成调试了。

在完成Atlas编译以后，就可以进行Atlas的安装了。Atlas的安装主要是安装Atlas的Server端，也就Atlas的管理页面，并确保Atlas与Kafka Hbase Solr等组件的集成。

在确保 底层存储与UI界面正常后，之后就可以进行与Hive等组件的集成调试了。

**1、环境准备**

安装之前 先要准备好

**JDK1.8**

**Zookeeper**

**Kafka**

**Hbase**

**Solr**

在启动Atlas时会配置这些环境变量的地址，所以一定要确保以上组件正常运行。

由于在编译时可以选择内部集成，所以这些Atlas是可以自带的，但是JDK一定要安装好。

在安装Altas中，需要Solr 预先创建好collection

```mipsasm
 bin/solr create -c vertex_index -shards 3 -replicationFactor 2

 bin/solr create -c edge_index -shards 3 -replicationFactor 2

bin/solr create -c fulltext_index -shards 3 -replicationFactor 2
```

在solr中验证创建成功。

**2、安装Atlas**

到编译好的包的路径下 apache-atlas-sources-2.1.0/distro/target

将生成好的安装包 apache-atlas-2.1.0-server.tar.gz 拷贝到目标路径下。

解压：

```undefined
tar -zxvf apache-atlas-2.1.0-server.tar.gz
```

**3、修改配置**

进入conf目录下：

```mipsasm
vi  atlas-env.sh 
```

在此指定JAVA_HOME和是否要用内嵌启动

```javascript
export JAVA_HOME=/opt/jdk1.8.0_191/
export MANAGE_LOCAL_HBASE=true
export MANAGE_LOCAL_SOLR=true  
```

如果使用内嵌，那么配置结束，直接去 **启动Atlas**

但是大部分时候，需要使用已经有的组件进行集成，所以设置为false。

```bash
export JAVA_HOME=/opt/jdk1.8.0_191/
export MANAGE_LOCAL_HBASE=false
export MANAGE_LOCAL_SOLR=false 
#注意修改Hbase配置文件路径
export HBASE_CONF_DIR=/opt/hbase/conf
```

修改其他配置

```matlab
vim atlas-application.properties 
```

这里就是设置Hbase Solr等配置

```ini
#Hbase地址  就是Hbase配置的zookeeper地址
atlas.graph.storage.hostname=slave01:2181,slave02:2181,slave03:2181

atlas.audit.hbase.zookeeper.quorum=slave01:2181,slave02:2181,slave03:2181

#solr服务器地址
atlas.graph.index.search.solr.http-urls=http://slave01:8984/solr

#kafka地址
atlas.notification.embedded=false
atlas.kafka.zookeeper.connect=slave01:2181,slave02:2181,slave03:2181
atlas.kafka.bootstrap.servers=slave01:9092,slave02:9092,slave03:9092

#atlas地址
atlas.rest.address=http://slave01:21000
```

### 4.5 启动Atlas

```bash
bin/atlas_start.py
```

启动成功后访问：http://slave01:21000，admin/admin登录

## 五、页面功能介绍

Atlas的页面功能非常的丰富，可以进行元数据的管理及数据血缘的展示。

### 5.1 主界面

![img](https://img-blog.csdnimg.cn/img_convert/0a54e8785d41ac73f7cee76c6cce2204.png)

### 5.2 Search

![img](https://img-blog.csdnimg.cn/img_convert/d474280fef9f9bdf7f97da395cfdfa73.png)

#### 基本搜索

基本搜索允许您使用实体的类型名称，关联的分类/标记进行查询，并且支持对实体属性以及分类/标记属性进行过滤。

可以使用 AND/OR 条件对多个属性进行基于属性的过滤。

![img](https://img-blog.csdnimg.cn/img_convert/b0c53607df82cbb40b5640c7d7cc2a38.png)
![img](https://img-blog.csdnimg.cn/img_convert/f08de2eb1d846f316314df6e201cbc29.png)
![img](https://img-blog.csdnimg.cn/img_convert/3fb7f9d12566a95f1974969f94c9db7b.png)

**支持的过滤运算符**

- LT（符号：<, lt）适用于数字、日期属性
- GT（符号：>、gt）适用于数字、日期属性
- LTE（符号：<=, lte）适用于数字、日期属性
- GTE（符号：>=，gte）适用于数字、日期属性
- EQ（符号：eq、=）适用于数字、日期、字符串属性
- NEQ（符号：neq、!=）适用于数字、日期、字符串属性
- LIKE（符号：like、LIKE）与字符串属性一起使用
- STARTS_WITH（符号：startsWith、STARTSWITH）与字符串属性一起使用
- ENDS_WITH（符号：endsWith、ENDSWITH）与字符串属性一起使用
- CONTAINS (symbols: contains, CONTAINS) 使用 String 属性

#### 高级搜索

Atlas 中的高级搜索也称为基于 DSL 的搜索。

领域特定搜索 (DSL) 是一种结构简单的语言，该语法模拟了关系数据库流行的结构化查询语言 (SQL)。

具体语法请参考[Github](https://github.com/apache/atlas/blob/master/repository/src/main/java/org/apache/atlas/query/antlr4/AtlasDSLParser.g4)上的Atlas DSL Grammer （Antlr G4格式）。

例：要检索名称可以是 time_dim 或 customer_dim 的 Table 类型的实体：

```sql
from Table where name = 'time_dim' or name = 'customer_dim'
```

### 5.3 Classification

- 分类传播使与实体相关联的分类能够自动与该实体的其他相关实体相关联。这在处理数据集从其他数据集派生数据的场景时非常有用 。

![img](https://img-blog.csdnimg.cn/img_convert/8ee6a917149ba4ecb35993bd3f44cb1e.png)

为实体添加分类

将分类“PII”添加到“hdfs_path”实体后，该分类将传播到沿袭路径中的所有受影响实体，包括“员工”表、视图“us_employees”和“uk_employees” - 如下所示。

![img](https://img-blog.csdnimg.cn/img_convert/de25cf579b4d8de91f4e3fcf18bf651f.png)

更新与实体关联的分类

与实体关联的分类的任何更新也将在分类传播到的所有实体中看到。

![img](https://img-blog.csdnimg.cn/img_convert/1b6e1a9e5a57d494d15e6fcb7f8ff077.png)

简单的说，此功能可以监控数据到底流向了哪里。

### 5.4 glossary

词汇表，也称术语表为业务用户提供适当的词汇表，它允许术语（词）相互关联并分类，以便在不同的上下文中理解它们。然后可以将这些术语映射到数据库、表、列等资产。这有助于抽象与存储库相关的技术术语，并允许用户发现/使用他们更熟悉的词汇表中的数据。

![img](https://img-blog.csdnimg.cn/img_convert/6ed0ba65e32ae026246d189c9340b9cc.png)

通过单击词汇表 UI 中的术语名称，可以查看术语的各种详细信息。详细信息页面下的每个选项卡提供该术语的不同详细信息。

![img](https://img-blog.csdnimg.cn/img_convert/3c90098169301a3e4b9988514180259d.png)

当切换开关在类别上时，面板将列出所有词汇表以及类别层次结构。这是此视图下可能的交互的列表。

![img](https://img-blog.csdnimg.cn/img_convert/f201210ea490a143f041bed4c6c00f03.png)

如果一个术语具有分类，则该实体已被分配继承相同的分类。

![img](https://img-blog.csdnimg.cn/img_convert/07e08e3957bb3b0827392f71e6499705.png)

通过术语表的功能，让数据资产与业务系统建立了联系。

## 六、Hive数据血缘

### 6.1 Hive2兼容性问题

Atlas与Hive存在兼容性问题，本文基于Atlas2.1.0兼容CDH6.3.2部署。Hive版本为2.1.1.其他版本的问题不在此文档讨论。

为兼容Hive2.1.1，需要修改源码重新编译。

- 所需修改的项目位置：`apache-atlas-sources-2.1.0\addons\hive-bridge`

①.org/apache/atlas/hive/bridge/HiveMetaStoreBridge.java 577行

```java
String catalogName = hiveDB.getCatalogName() != null ? hiveDB.getCatalogName().toLowerCase() : null;
```

改为：

```java
String catalogName = null;
```

②.org/apache/atlas/hive/hook/AtlasHiveHookContext.java 81行

```java
this.metastoreHandler = (listenerEvent != null) ? metastoreEvent.getIHMSHandler() : null;
```

改为：C:\Users\Heaton\Desktop\apache-atlas-2.1.0-sources\apache-atlas-sources-2.1.0\addons

```java
this.metastoreHandler = null;
```

### 6.2 集成Hive

- 将 atlas-application.properties 配置文件，压缩加入到 atlas-plugin-classloader-2.0.0.jar 中

```bash
#必须在此路径打包，才能打到第一级目录下
cd /usr/local/src/atlas/apache-atlas-2.1.0/conf

zip -u /usr/local/src/atlas/apache-atlas-2.1.0/hook/hive/atlas-plugin-classloader-2.1.0.jar atlas-application.properties
```

- 修改 hive-site.xml

![img](https://img-blog.csdnimg.cn/img_convert/1bf1aab5248fa447e98da9ed072edb1f.png)

```xml
<property>
    <name>hive.exec.post.hooks</name>
    <value>org.apache.atlas.hive.hook.HiveHook</value>
</property>
```

- 修改 hive-env.sh 的 Gateway 客户端环境高级配置代码段（安全阀）

![img](https://img-blog.csdnimg.cn/img_convert/95523bb7dd11349fd3079586c140a56e.png)

```java
HIVE_AUX_JARS_PATH=/usr/local/src/atlas/apache-atlas-2.1.0/hook/hive
```

- 修改 HIVE_AUX_JARS_PATH

![img](https://img-blog.csdnimg.cn/img_convert/b3365c7d308033315d27ab098da774cc.png)

- 修改 hive-site.xml 的 HiveServer2 高级配置代码段（安全阀）

![img](https://img-blog.csdnimg.cn/img_convert/f47d63efba1a45585d7ebaa6a11fa89a.png)

```xml
<property>
    <name>hive.exec.post.hooks</name>
    <value>org.apache.atlas.hive.hook.HiveHook</value>
</property>
<property>
    <name>hive.reloadable.aux.jars.path</name>
    <value>/usr/local/src/atlas/apache-atlas-2.1.0/hook/hive</value>
</property>
```

- 修改 HiveServer2 环境高级配置代码段

![img](https://img-blog.csdnimg.cn/img_convert/8498e58a422ce155ecc95a1daaa0b911.png)

```bash
HIVE_AUX_JARS_PATH=/usr/local/src/atlas/apache-atlas-2.1.0/hook/hive
```

需要将配置好的Atlas包发往各个hive节点后重启集群。

### 6.3 导入Hive元数据

- 执行atlas脚本

```java
./bin/import-hive.sh
#输入用户名：admin；输入密码：admin
```

登录Atlas查看元数据信息。

### 6.4 测试实时hive hook

简单执行一个hera资源统计的hive脚本

```sql
use sucx_test
;

-- 昨日升级设备数
create table if not exists qs_tzl_ProductTag_result(
pid string
,category string
,category_code string
,tag_name string
,tag_value string
,other string
,update_time string
)
partitioned by (tag_id string)
STORED AS PARQUET TBLPROPERTIES('parquet.compression'='SNAPPY')
;


insert overwrite table qs_tzl_ProductTag_result partition(tag_id='3014')
select
       T1.product_id as pid
       ,T2.category
       ,cast(from_unixtime(unix_timestamp()) as string) as update_time
from (select
             product_id
      from COM_PRODUCT_UPGRADE_STAT_D where p_day='20200901' 
     ) T1
left join (select category
        from bi_ods.ods_smart_product where dt='20200901'
        ) T2
on T1.product_id=T2.id
;
```

执行后查看 qs_tzl_ProductTag_result 的表级血缘为

![img](https://img-blog.csdnimg.cn/img_convert/e0224765c53a0cac7051dadafd90cb34.png)

pid 的字段血缘为

![img](https://img-blog.csdnimg.cn/img_convert/251a32be14b86d746cf6d9a9115fe569.png)

### 6.5 Hive Hook

**Atlas获取Hive元数据主要有两步：**

- ①**在Hive中设置Atlas Hook** ：

  Atlas Hive hook向Hive注册以**监听创建/更新/删除操作**，并**通过Kafka`实时通知更新Atlas中的元数据`，以感知Hive中的更改**。

  ```xml
  <property>
      <name>hive.exec.post.hooks</name>
        <value>org.apache.atlas.hive.hook.HiveHook</value>
    </property>
  ```

- ②**导入Hive元数据**

  Apache Atlas提供了一个命令行脚本`import-hive.sh`，用于将Apache Hive数据库和表的元数据导入Apache Atlas。**该脚本可用于使用Apache Hive中的数据库/表`初始化`Apache Atlas**。此脚本支持导入特定表的元数据，特定数据库中的表或所有数据库和表。

- 其他类型Hook配置参考：https://atlas.apache.org/1.1.0/index.html

## 七、Spark数据血缘

### 7.1 打包spark-atlas-connector

`atlas` 官方文档中并不支持 `spark sql` 的解析，需要使用第三方的包。

地址： https://github.com/hortonworks-spark/spark-atlas-connector

一、git clone 后本地进行打包

`mvn package -DskipTests`

二、打包后在 spark-atlas-connector/spark-atlas-connector-assembly/target 目录有一个 spark-atlas-connector-assembly-${version}.jar 的 jar，将该 jar 上传到服务器。

需要注意的是不要上传 spark-atlas-connector/spark-atlas-connector/target 这个目录内的 jar ，缺少相关依赖包

三、将 spark-atlas-connector-assembly-${version}.jar 放到一个固定目录 比如/opt/resource

### 7.2 测试spark hook

首先进入spark-sql client

```sql
spark-sql --master yarn  \
--jars /opt/resource/spark-atlas-connector_2.11-0.1.0-SNAPSHOT.jar  \
--files /opt/resource/atlas-application.properties \
--conf spark.extraListeners=com.hortonworks.spark.atlas.SparkAtlasEventTracker \
--conf spark.sql.queryExecutionListeners=com.hortonworks.spark.atlas.SparkAtlasEventTracker \
--conf spark.sql.streaming.streamingQueryListeners=com.hortonworks.spark.atlas.SparkAtlasStreamingQueryEventTracker
执行 hera 的一个资源统计任务

CREATE EXTERNAL TABLE  IF NOT EXISTS sucx_hera.ads_hera_task_mem_top_10(
    `job_id` BIGINT COMMENT '任务ID',
    `user` STRING COMMENT '关注人',
    `applicationId` STRING COMMENT 'yarn执行的app id',
    `memorySeconds` BIGINT COMMENT '内存使用时间和',
    `startedTime` BIGINT COMMENT '开始时间',
    `finishedTime` BIGINT COMMENT '结束时间',
    `elapsedTime` BIGINT COMMENT '运行时间',
    `vcoreSeconds` BIGINT COMMENT 'vcore使用时间和')

ROW FORMAT SERDE
    'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
WITH SERDEPROPERTIES (
    'field.delim'='\t',
    'serialization.format'='\t')
STORED AS INPUTFORMAT
    'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT
    'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
    'cosn://sucx-big-data/bi//sucx_hera/ads_hera_task_mem_top_10';

 

insert overwrite table sucx_hera.ads_hera_task_mem_top_10

select
 job_id,user,applicationId,memorySeconds,startedTime,finishedTime,elapsedTime,vcoreSeconds
from
(SELECT
    top.job_id,
    row_number() over(distribute by top.applicationId ORDER BY sso.id) as num,
    case when sso.name is null then operator
    else sso.name end as user,
    top.applicationId,
    top.memorySeconds,
    top.startedTime,
    top.finishedTime,
    top.elapsedTime,
    top.vcoreSeconds

FROM (
select * from sucx_hera.dws_hera_task_mem_top_10 where dt = '20200901' ) top

left join bi_ods.ods_hera_job_monitor monitor

on  monitor.dt='20200901' and top.job_id=monitor.job_id

left join bi_ods.ods_hera_sso sso

on sso.dt='20200901' and find_in_set(sso.id,monitor.user_ids) >0 order by job_id ) temp

where temp.num = 1
```

执行后，查看 `ads_hera_task_mem_top_10` 表级血缘

![img](https://img-blog.csdnimg.cn/img_convert/42b44a325925ad271ff6d2daacc1e291.png)

注意此包不支持 `spark` 字段的支持。

如果需要spark字段的支持，一种是spark代码转成hive跑一遍，一种就是需要自研了。

## 八、Atlas二次开发

atlas虽好，但是很多场景依然无法满足我们的需要。这时候就不得不做一些改动了。二次开发有两种方式一种是基于Atlas Api的开发，一种是修改源码。

### 8.1 REST API

http://atlas.apache.org/api/v2/index.html

DiscoveryREST

```bash
http://hostname:21000/api/atlas/v2/search/basic?classification=class1
参数支持：query、typeName、classification、excludeDeletedEntities、limit、offset
code：https://github.com/apache/atlas/blob/6bacbe946bbc5ca72118304770d5ad920695bd52/webapp/src/main/java/org/apache/atlas/web/rest/DiscoveryREST.java

# 查询所有 Table
http://hostname:21000/api/atlas/v2/search/dsl?typeName=Table

# 查询 owner 前缀为 John 的 Table
http://hostname:21000/api/atlas/v2/search/attribute?typeName=Table&attrName=owner&attrValuePrefix=John

# 查询 Table 的一些属性，如：guid, ownerName, searchParameters 等
http://hostname:21000/api/atlas/v2/search/saved/Table

# 不是很懂（返回的数据和上面那个 API 一模一样）
http://hostname:21000/api/atlas/v2/search/saved

# 查询 EntityType - Table 下有哪些 entity.
http://hostname:21000/api/atlas/v2/search/saved/execute/Table

# 查询 guid 为 e283d8c1-ae19-4f4b-80c0-38031788383b 的 EntityType 下有哪些 entity.
http://hostname:21000/api/atlas/v2/search/saved/execute/guid/e283d8c1-ae19-4f4b-80c0-38031788383b
```

LineageREST

```bash
# 查询 guid 为 a95cb57f-4643-4edf-b5a5-0f1de2e0f076 的实体的血缘
http://hostname:21000/api/atlas/v2/lineage/a95cb57f-4643-4edf-b5a5-0f1de2e0f076
```

EntityREST

```bash
# 查询 guid 为 48f29229-47a9-4b05-b053-91e6484f42a1 的实体
http://hostname:21000/api/atlas/v2/entity/guid/48f29229-47a9-4b05-b053-91e6484f42a1

# 查询 guid 为 48f29229-47a9-4b05-b053-91e6484f42a1 的实体的审计数据
http://hostname:21000/api/atlas/v2/entity/48f29229-47a9-4b05-b053-91e6484f42a1/audit

# 查询 guid 为 48f29229-47a9-4b05-b053-91e6484f42a1 的实体所属的 classifications
http://hostname:21000/api/atlas/v2/entity/guid/48f29229-47a9-4b05-b053-91e6484f42a1/classifications

# 根据 EntityType 的唯一属性查找实体
# 如下：name 是 DB 的一个唯一属性. 查找 name=Logging 的 DB.
http://hostname:21000/api/atlas/v2/entity/uniqueAttribute/type/DB?attr:name=Logging

# 查询 entity 与 classification 是否有关？
http://hostname:21000/api/atlas/v2/entity/guid/48f29229-47a9-4b05-b053-91e6484f42a1/classification/Dimension

# 批量查询实体
http://hostname:21000/api/atlas/v2/entity/bulk?guid=e667f337-8dcc-468b-a5d0-96473f8ede26&guid=a95cb57f-4643-4edf-b5a5-0f1de2e0f076
```

RelationshipREST

```bash
# 查询 guid 为 726c0120-19d2-4978-b38d-b03124033f41 的 relationship
# 注：relationship 可以看做是血缘的一条边
http://hostname:21000/api/atlas/v2/relationship/guid/726c0120-19d2-4978-b38d-b03124033f41
```

TypesREST

```bash
http://hostname:21000/api/atlas/v2/types/typedef/guid/e0ca4c40-6165-4cec-b489-2b8e5fc7112b
http://hostname:21000/api/atlas/v2/types/typedef/name/Table
http://hostname:21000/api/atlas/v2/types/typedefs/headers
http://hostname:21000/api/atlas/v2/types/typedefs
http://hostname:21000/api/atlas/v2/types/enumdef/name/hive_principal_type
http://hostname:21000/api/atlas/v2/types/enumdef/guid/ee30446a-92e1-4bbc-aa0a-66ac21970d88
http://hostname:21000/api/atlas/v2/types/structdef/name/hive_order
http://hostname:21000/api/atlas/v2/types/structdef/guid/0b602605-8c88-4b60-a177-c1c671265294
http://hostname:21000/api/atlas/v2/types/classificationdef/name/PII
http://hostname:21000/api/atlas/v2/types/classificationdef/guid/3992eef8-fd74-4ae7-8b35-fead363b2122
http://hostname:21000/api/atlas/v2/types/entitydef/name/DataSet
http://hostname:21000/api/atlas/v2/types/entitydef/guid/5dca20ce-7d57-4bc3-a3f1-93fa622c32b1
http://hostname:21000/api/atlas/v2/types/relationshipdef/name/process_dataset_outputs
http://hostname:21000/api/atlas/v2/types/relationshipdef/guid/5d76dae0-6bad-4d19-b9b2-cb0cdc3b53d5
```

GlossaryREST

```bash
http://hostname:21000/api/atlas/v2/glossary
http://hostname:21000/api/atlas/v2/glossary/302dc8f8-3bc8-4878-9ba2-0c585ca6de3d
```

### 8.2 本地开发环境

**启动HBase和solr**

由于在`Atlas`需要使用到`HBase`以及`slor`。方便调试可以解压之前编译好内置`HBase和sole`的tar。

**文件补充**

创建文件夹将，atlas所需要的文件拷贝过去。

将编译内置HBase以及solr源码目录distro/target/conf下的文件拷贝到上述的上图的conf目录。

将源码中addons/models下所有的文件拷贝过去。

将源码中的atlas/webapp/target中的内容全部拷贝到webapp目录中。

**源码启动**

将clone下来的源码导入到IDEA中。配置启动参数：

```ini
-Datlas.home=/opt/deploy
-Datlas.conf=/opt/deploy/conf
-Datlas.data=/opt/deploy/data
```

## 附录A

## A.1 参考文章

- http://atlas.apache.org/
- [atlas 初体验](https://blog.csdn.net/su20145104009/article/details/108253556)
- [环境篇：Atlas2.1.0兼容CDH6.3.2部署](https://www.cnblogs.com/ttzzyy/p/14143508.html)
- [数据治理---Apache Atlas元数据管理](https://blog.csdn.net/qq_35995514/article/details/107395181)
- [Apache Atlas数据治理和元数据框架](https://blog.csdn.net/lightupworld/article/details/108501201)
- [元数据管理atlas导入hive和hbase元数据以及生成血缘](https://blog.csdn.net/hblicy/article/details/106890270)
- [别人家的元数据系统是怎么设计的](https://cloud.tencent.com/developer/article/1135920)
- [基于Apache doris的元数据管理系统](https://zhuanlan.zhihu.com/p/404234904)

### A.2 源码分析

- [Apache-atlas-kafka-hook-源码分析](https://blog.csdn.net/windydreams/article/details/127813483)
- [apache-atlas-hive-hook-源码分析](https://blog.csdn.net/windydreams/article/details/127893478)
- [apache-atlas-hive-bridge-源码分析](https://blog.csdn.net/windydreams/article/details/127843534)
- [apache-atlas-hbase-hook源码分析](https://blog.csdn.net/windydreams/article/details/128013418)
- [apache-atlas-hbase-bridge-源码分析](https://blog.csdn.net/windydreams/article/details/127978562)