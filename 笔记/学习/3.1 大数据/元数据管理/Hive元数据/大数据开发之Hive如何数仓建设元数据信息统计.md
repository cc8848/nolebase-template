- [大数据开发之Hive如何数仓建设元数据信息统计](https://baijiahao.baidu.com/s?id=1713565462025429114)
- [hive 元数据信息表解析](https://blog.csdn.net/qq_44326412/article/details/127281079)

在数据仓库建设中，元数据管理是非常重要的环节之一。根据Kimball的数据仓库理论，可以将元数据分为这三类：

- 技术元数据，如表的存储结构结构、文件的路径

- 业务元数据，如血缘关系、业务的归属

- 过程元数据，如表每天的行数、占用HDFS空间、更新时间

而基于这3类元数据"搭建"起来的元数据系统，通常又会实现如下核心功能：

- 血缘关系：如表级别/字段级别的血缘关系，这些主要体现在我们日常的SQL和ETL任务里。

**大数据集群计算资源管理：**

针对利用不同的计算引擎如Spark/Flink/Mapreduce，可以到Yarn（也可能是其他资源管理器）上采集相关任务的使用情况。如CPU、内存、磁盘IO使用情况。

然后可以把这些资源使用情况绘制成图。通过可视化界面可以直观发现某些任务中的异常情况，以及发现某些严重消耗资源的表或业务，及时通知相关负责人有针对性的分析处理和优化。

**Hive库表元数据信息统计：**

这里对Hive库表统计信息主要是指：行数、文件数、所占HDFS存储大小、最后一次操作时间等。

通过持续不断的采集这些指标，形成可视化曲线图，数据仓库相关人员都可以从这个图中发现数据规律或数据质量问题。对于利用数仓进行业务开发的人员，可以通过这些曲线图来分析业务量变化趋势。在此基础之上，还可以做数据质量校验、数值分布探查等功能。

本文主要介绍如何利用Hive和Spark进行对Hive库、分区表/非分区表相关指标的统计。

而在我们实际生产中，我们不仅可以通过如下的方式及时更新和获取Hive元数据库中相关表记录的指标信息，我们也可以参考下述相关SQL在Hive/Spark底层的执行过程，实现我们自己的一整套业务逻辑。

# 一、Hive元数据库中主要涉及的元数据表

DBS：存储Hive中所有数据库的基本信息，如库ID、表ID、创建时间、用户、表名、表的类型等。

字段如下:

| 表字段          | 说明               | 示例数据                                |
| --------------- | ------------------ | --------------------------------------- |
| DB_ID           | 数据库ID           | 1                                       |
| DESC            | 数据库描述         | Default  Hive database                  |
| DB_LOCATION_URI | 数据HDFS路径       | hdfs://193.168.1.75:9000/test-warehouse |
| NAME            | 数据库名           | default                                 |
| OWNER_NAME      | 数据库所有者用户名 | public                                  |
| OWNER_TYPE      | 所有者角色         | ROLE                                    |

TBS：存储Hive表、视图等的基本信息，如表ID、表名、创建时间、用户、表类型等。

主要有**TBLS、TABLE_PARAMS、TBL_PRIVS**，这三张表通过TBL_ID关联。

**metastore.TBLS:**该表中存储Hive表，视图，索引表的**基本信息，**主要包括表id、表名、表类型、视图Hsql等

| 表字段             | 说明              | 示例数据                   |
| ------------------ | ----------------- | -------------------------- |
| TBL_ID             | 表ID              | 21                         |
| CREATE_TIME        | 创建时间          | 1447675704                 |
| DB_ID              | 数据库ID          | 1                          |
| LAST_ACCESS_TIME   | 上次访问时间      | 1447675704                 |
| OWNER              | 所有者            | root                       |
| RETENTION          | 保留字段          | 0                          |
| SD_ID              | 序列化配置信息    | 41，对应SDS表中的SD_ID     |
| TBL_NAME           | 表名              | ex_detail_ufdr_30streaming |
| TBL_TYPE           | 表类型            | EXTERNAL_TABLE             |
| VIEW_EXPANDED_TEXT | 视图的详细HQL语句 |                            |
| VIEW_ORIGINAL_TEXT | 视图的原始HQL语句 |                            |

TABLE_PARAMS：存储表等的属性信息，表ID、PARAM_KEY（如EXTERNAL）、PARAM_VALUE（与PARAM_KEY对应的值）。

字段如下:

| 表字段          | 说明               | 示例数据                                |
| --------------- | ------------------ | --------------------------------------- |
| DB_ID           | 数据库ID           | 1                                       |
| DESC            | 数据库描述         | Default  Hive database                  |
| DB_LOCATION_URI | 数据HDFS路径       | hdfs://193.168.1.75:9000/test-warehouse |
| NAME            | 数据库名           | default                                 |
| OWNER_NAME      | 数据库所有者用户名 | public                                  |
| OWNER_TYPE      | 所有者角色         | ROLE                                    |

PARTITIONS：存储Hive分区统计信息相关的元数据，如分区ID、表ID、创建时间、分区名（partCol=partVal）等信息。

PARTITION_PARAMS：存储Hive分区统计信息相关的元数据，如分区ID、PARAM_KEY（如文件数）、PARAM_VALUE（与

PARAM_KEY对应的值）。

# 二、Hive和Spark支持的Hive库表元数据信息统计

## 2.1 Hive

### 2.1.1 语法支持

默认情况下，在对Hive表进行数据insert时，会自动更新元数据库表中的统计信息，但主要是文件数、占用HDFS空间大小等，不包括行数。

1）分区表

Hive分区表元数据统计信息SQL语法需要指定到具体分区，如分区字段或者分区名=分区值

```sql
-- 1. 统计更新tab_partition的分区字段为dt的所有元数据信息
analyze table tab_partition partition(dt) COMPUTE STATISTICS;
-- 2. 统计更新单个分区元数据统计信息
analyze table tab_partition partition(dt='20200722000000') COMPUTE STATISTICS;
-- 在Hive shell中执行analyze时，如果进行了元数据信息统计会打印类似如下信息：
Partition default.test_partition2{dt=20200718000000} stats: [numFiles=1, numRows=2, totalSize=418, rawDataSize=6]
```

2）非分区表

```sql
-- 非分区表粒度到表
analyze table tab_no_partition COMPUTE STATISTICS;
```

### 2.1.2 Hive元数据库中涉及的元数据统计信息字段

1）Hive分区表

-- 表级别：TABLE_PARAMS

-- Hive分区级别：PARTITION_PARAMS

numFiles：文件数

numRows：行数

totalSize：占用HDFS空间大小

rawDataSize：原生数据大小

transient_lastDdlTime：最近一次操作时间

2）Hive非分区表

对于Hive分区表，因为最小粒度是表级别。因此，元数据统计信息也是表级别的。

-- TABLE_PARAMS

numFiles、numRows、totalSize、rawDataSize、transient_lastDdlTime：含义同上

## 2.2 Spark

注意：Spark默认不统计文件数

### 2.2.1 语法支持

1）分区表

Spark对Hive分区表元数据统计，跟Hive原生对分区表的统计支持略有不同。

Spark既支持具体到分区的元数据信息统计，也支持整个表级别的元数据信息统计（但不会对具体分区做处理）

```sql
-- 统计tab_partition数据所占HDFS空间总大小和总行数。
-- Hive目前不支持直接这样解析分区表
-- 注意：执行该SQL不会处理表中具体分区统计信息
analyze table tab_partition COMPUTE STATISTICS;
-- 同Hive
analyze table tab_partition partition(partCol) COMPUTE STATISTICS;
-- 同Hive
analyze table tab_partition partition(partCol='20200722000000') COMPUTE STATISTICS;
```

2）非分区表

```sql
analyze table tab_no_partition COMPUTE STATISTICS;
```

### 2.2.2 Hive元数据库中涉及的元数据统计信息字段

1）Hive分区表

-- 表级别：TABLE_PARAMS

-- Hive分区级别：PARTITION_PARAMS

spark.sql.statistics.numRows：文件数（同Hive统计中的numRows，但不会更新Hive的统计信息）

spark.sql.statistics.totalSize：行数（同Hive统计中的totalSize，但不会更新Hive的统计信息）

transient_lastDdlTime：同Hive

2）Hive非分区表

```sql
-- 统计级别同Hive，TABLE_PARAMS
spark.sql.statistics.numRows、spark.sql.statistics.totalSize、transient_lastDdlTime：含义同上
```

# 三、Hive和Spark对Hive库表元数据信息统计的主要区别

对Hive表元数据信息统计的SQL语法支持不同

如Spark支持对Hive分区表进行表级别的统计，但Hive需要指定到具体分区

对Hive表元数据信息统计在Hive元数据库中的体现不同

如同样是行数，Hive用numRows，而Spark用spark.sql.statistics.numRows

Spark默认不统计文件数，但Hive统计

Hive和Spark对Hive库表元数据信息统计的区别包括但不限于以上3种区别。具体的看之前的介绍，以及通过下面以Hive分区表为例，看看主要的具体细节：

## 3.1 Hive

默认情况下，在对Hive表进行数据insert时，Hive会自动更新元数据统计信息，但是不统计行数。如需获取numRow，可以再次执行analyze SQL

1）直接通过Hive进行表的创建

以分区表testdb.test_analyze为例，表刚创建时Hive元数据库中表TABLE_PARAMS的信息：

```sql
+------+---------------------+-----------+
|TBL_ID| PARAM_KEY |PARAM_VALUE|
+------+---------------------+-----------+
| 3016| EXTERNAL | TRUE|
| 3016|transient_lastDdlTime| 1595405772|
+------+---------------------+-----------+
```

2）对表testdb.test_analyze进行数据的保存和元数据信息统计：

```sql
insert overwrite table testdb.test_analyze partition(partCol=20200721000000) select id,name from testdb.test_partition1 where partCol=20190626000000;
analyze table testdb.test_analyze partition(partCol='20200721000000') COMPUTE STATISTICS;
```

3）连接Hive元数据库，查询testdb.test_analyze的元数据统计信息

```sql
-- 1. 连接Hive元数据库
connect jdbc where
url="jdbc:mysql://localhost:3306/hive?useUnicode=true&amp;characterEncoding=UTF-8"
and driver="com.mysql.jdbc.Driver"
and user="root"
and password="root"
as db_1;
-- 2. 将TABLE_PARAMS、DBS、TBLS、PARTITIONS、PARTITION_PARAMS注册为临时表
-- load jdbc.`db_1.TABLE_PARAMS` as TABLE_PARAMS ;
load jdbc.`db_1.DBS` as dbs;
load jdbc.`db_1.TBLS` as tbls;
load jdbc.`db_1.PARTITIONS` as partitions;
load jdbc.`db_1.PARTITION_PARAMS` as partition_params;
-- 3. 获取testdb.test_analyze的元数据统计信息
select d.NAME,t.TBL_NAME,t.TBL_ID,p.PART_ID,p.PART_NAME,a.*
from tbls t
left join dbs d
on t.DB_ID = d.DB_ID
left join partitions p
on t.TBL_ID = p.TBL_ID
left join partition_params a
on p.PART_ID=a.PART_ID
where t.TBL_NAME='test_analyze' and d.NAME='testdb';
```

4）结果

```sql
-- 测试时，testdb.test_analyze只有partCol=20200721000000的分区。因此，统计信息也只有partCol=20200721000000的
+------+------------+------+-------+----------------------+-------+--------------------+--------------------+
| NAME| TBL_NAME|TBL_ID|PART_ID| PART_NAME|PART_ID| PARAM_KEY| PARAM_VALUE|
+------+------------+------+-------+----------------------+-------+--------------------+--------------------+
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976|COLUMN_STATS_ACCU...|{"BASIC_STATS":"t...|
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976| numFiles| 1|
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976| numRows| 1|
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976| rawDataSize| 3|
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976| totalSize| 383|
|testdb|test_analyze| 3016| 52976|partCol=20200721000000| 52976|transient_lastDdl...| 1595407507|
+------+------------+------+-------+----------------------+-------+--------------------+--------------------+
```

## 3.2 Spark

1）通过Spark创建Hive表

以分区表testdb.test_analyze_spark为例，表刚创建时Hive元数据库中表TABLE_PARAMS的信息：

```sql
+------+------------------------------------+--------------------+
|TBL_ID| PARAM_KEY| PARAM_VALUE|
+------+------------------------------------+--------------------+
| 3018| EXTERNAL| TRUE|
| 3018| spark.sql.create.version| 2.4.3|
| 3018|spark.sql.sources.schema.numPartCols| 1|
| 3018| spark.sql.sources.schema.numParts| 1|
| 3018| spark.sql.sources.schema.part.0|{"type":"struct",...|
| 3018| spark.sql.sources.schema.partCol.0| dt|
| 3018| transient_lastDdlTime| 1595409374|
+------+------------------------------------+--------------------+
```

2）对表testdb.test_analyze进行数据的保存和元数据信息统计

```sql
insert overwrite table testdb.test_analyze partition(partCol=20200721000000) select id,name from testdb.test_partition1 where partCol=20190626000000;
```

执行上述SQL后，Hive内部会启动一个任务进行Hive表操作的分区元数据信息统计，但是没有numRows。如下：

```sql
+------+------------------+------+-------+----------------------+-------+--------------------+-----------+
| NAME| TBL_NAME|TBL_ID|PART_ID| PART_NAME|PART_ID| PARAM_KEY|PARAM_VALUE|
+------+------------------+------+-------+----------------------+-------+--------------------+-----------+
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| numFiles| 1|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| totalSize| 389|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977|transient_lastDdl...| 1595409909|
+------+------------------+------+-------+----------------------+-------+--------------------+-----------+
```

3）连接Hive元数据库，查询testdb.test_analyze_spark的元数据统计信息

```sql
connect jdbc where
url="jdbc:mysql://localhost:3306/hive?useUnicode=true&amp;characterEncoding=UTF-8"
and driver="com.mysql.jdbc.Driver"
and user="root"
and password="root"
as db_1;
-- load jdbc.`db_1.TABLE_PARAMS` as TABLE_PARAMS ;
load jdbc.`db_1.TBLS` as tbls;
load jdbc.`db_1.DBS` as dbs;
load jdbc.`db_1.PARTITIONS` as partitions;
load jdbc.`db_1.PARTITION_PARAMS` as partition_params;
select d.NAME,t.TBL_NAME,t.TBL_ID,p.PART_ID,p.PART_NAME,a.*
from tbls t
left join dbs d
on t.DB_ID = d.DB_ID
left join partitions p
on t.TBL_ID = p.TBL_ID
left join partition_params a
on p.PART_ID=a.PART_ID
where t.TBL_NAME='test_analyze_spark' and d.NAME='testdb' ;
```

4）结果

```sql
-- Spark在执行analyze table mlsql_test.test_analyze_spark partition(dt='20200721000000') COMPUTE STATISTICS; 时，会对分区行数进行统计：
+------+------------------+------+-------+----------------------+-------+-------------------------------+-----------+
| NAME| TBL_NAME|TBL_ID|PART_ID| PART_NAME|PART_ID| PARAM_KEY|PARAM_VALUE|
+------+------------------+------+-------+----------------------+-------+-------------------------------+-----------+
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| numFiles| 1|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| spark.sql.statistics.numRows| 1|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| spark.sql.statistics.totalSize| 389|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| totalSize| 389|
|testdb|test_analyze_spark| 3018| 52977|partCol=20200721000000| 52977| transient_lastDdlTime| 1595410238|
+------+------------------+------+-------+----------------------+-------+-------------------------------+-----------+
```

5）通过Spark对整个Hive分区表元数据信息的统计

```sql
-- 1. 执行：analyze table testdb.test_analyze_spark COMPUTE STATISTICS;
-- 2. Hive元数据库中表TABLE_PARAMS的包含的testdb.test_analyze_spark信息：
connect jdbc where
url="jdbc:mysql://localhost:3306/hive?useUnicode=true&amp;characterEncoding=UTF-8"
and driver="com.mysql.jdbc.Driver"
and user="root"
and password="root"
as db_1;
-- 获取mlsql_test的DB_ID（49）
load jdbc.`db_1.DBS` as dbs;
select DB_ID from dbs where NAME='testdb' as db;
-- 获取test_analyze_spark的TBL_ID（3018）
load jdbc.`db_1.TBLS` as tbls;
select TBL_ID from tbls where DB_ID=49 and TBL_NAME='test_analyze_spark' as t2;
-- 获取testdb.test_analyze_spark表级别统计信息
load jdbc.`db_1.TABLE_PARAMS` as TABLE_PARAMS ;
select * from TABLE_PARAMS where TBL_ID=3018 ;
-- 结果
+------+------------------------------------+--------------------+
|TBL_ID| PARAM_KEY| PARAM_VALUE|
+------+------------------------------------+--------------------+
| 3018| EXTERNAL| TRUE|
| 3018| spark.sql.create.version| 2.4.3|
| 3018|spark.sql.sources.schema.numPartCols| 1|
| 3018| spark.sql.sources.schema.numParts| 1|
| 3018| spark.sql.sources.schema.part.0|{"type":"struct",...|
| 3018| spark.sql.sourc
es.schema.partCol.0| partCol|
| 3018| spark.sql.statistics.numRows| 1|
| 3018| spark.sql.statistics.totalSize| 389|
| 3018| transient_lastDdlTime| 1595410958|
+------+------------------------------------+--------------------+
```

# 四、Hive-元数据查询表数据量

- [Hive-元数据查询表数据量](https://blog.csdn.net/docsz/article/details/118364840)

## 4.1 Statistics使用

### 4.1.1 表，分区级别的统计信息

对于新创建的表或者分区，默认情况下，如果通过INSERT OVERWRITE的方式插入数据，那么Hive会自动将该表或分区的统计信息更新到元数据。有一个参数来控制是否自动统计，hive.stats.autogather，默认为true.

对于已经存在表或分区我们可以通过ANALYZE命令手动更新其Statistics信息：

```sql
ANALYZE TABLE Table1 COMPUTE STATISTICS;//统计全表的所有分区的信息

ANALYZE TABLE Table1 COMPUTE STATISTICS NOSCAN;//只统计文件数和文件大小，不扫描文件行数，执行较快

ANALYZE TABLE Table1 PARTITION(ds='2008-04-09', hr=11) COMPUTE STATISTICS;//统计执行分区的信息
```

### 4.1.2 统计Hive元信息

- 非分区表的statics信息存在hive元数据表TABLE_PARAMS中

```sql
select * from TABLE_PARAMS where TBL_ID=102966
```

- 分区表的statics信息存在hive元数据表PARTITION_PARAMS 中

```sql
select * from PARTITION_PARAMS where PART_ID=3011
```

## 4.2 列级别的统计信息

对于列级别的统计信息默认自动统计是关闭的，由参数hive.stats.column.autogather控制，默认false(hive3.0后为true)。

同样我们可以通过ANALYZE命令手动更新其Statistics信息：

```sql
ANALYZE TABLE Table1 COMPUTE STATISTICS FOR COLUMNS;//统计全表或所有分区列信息
ANALYZE TABLE Table1 PARTITION(ds, hr) COMPUTE STATISTICS FOR COLUMNS;//统计指定分区列的信息
```

- 对于非分区表列的statics信息存在hive元数据表TABLE_COL_STATS中
- 对于分区表列的statics信息存在hive元数据表PART_COL_STATS中

## 4.3 查询

与关系库一样hive表也能够通过查询元数据来得到总条数：

```sql
select d.NAME,t.TBL_NAME,t.TBL_ID,p.PART_ID,p.PART_NAME,a.PARAM_VALUE 
from TBLS t 
left join DBS d
on t.DB_ID = d.DB_ID
left join PARTITIONS p
on t.TBL_ID = p.TBL_ID 
left join PARTITION_PARAMS a
on p.PART_ID=a.PART_ID
where t.TBL_NAME='emp' and d.NAME='ods' and a.PARAM_KEY='numRows';

select FORMAT(sum(a.PARAM_VALUE),0)
from TBLS t 
left join DBS d
on t.DB_ID = d.DB_ID
left join PARTITIONS p
on t.TBL_ID = p.TBL_ID 
left join PARTITION_PARAMS a
on p.PART_ID=a.PART_ID
where t.TBL_NAME='emp' and d.NAME='ods' and a.PARAM_KEY='numRows';
```

## 4.4 使用shell命令查看表存储文件大小

```powershell
-- 单位b
hadoop fs -ls  /user/hive/warehouse/dwd.db/comanpy/batch_no=97 | awk -F ' ' '{print $5}'|awk '{a+=$1}END {print a}'
--#查看分区表的容量 单位GB

hadoop fs -ls  /user/hive/warehouse/dwd.db/comanpy/batch_no=97 | awk -F ' ' '{print $5}'|awk '{a+=$1}END {print a/(1024*1024*1024)}'

-- 自动转换单位
hadoop fs -du /user/hive/warehouse/dwd.db/comanpy/batch_no=97 | awk '{ sum=$1 ;dir2=$2 ; hum[1024**3]="Gb";hum[1024**2]="Mb";hum[1024]="Kb"; for (x=1024**3; x>=1024; x/=1024){ if (sum>=x) { printf "%.2f %s \t %s\n",sum/x,hum[x],dir2;break } }}'
```

## 4.5 查看Hive表

```sql
desc extended dwd.comanpy;//表字段
show create table dwd.comanpy;//建表语句
show tblproperties dwd.comanpy;//表属性
```

