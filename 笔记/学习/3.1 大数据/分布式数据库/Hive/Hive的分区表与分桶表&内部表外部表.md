- [Hive的分区表与分桶表概念和使用场景](https://www.studytime.xin/article/hive-partition-and-bucket.html)
- [Hive建表语句、内部表vs外部表、加载数据](https://www.cnblogs.com/saowei/p/15911652.html)
- [Hive分区和Hive动态分区](https://www.cnblogs.com/saowei/p/15915011.html)
- [Hive分桶、Hive JDBC](https://www.cnblogs.com/saowei/p/15915609.html)
- [Hive数据类型](https://www.cnblogs.com/saowei/p/15916237.html)
- [Hive HQL语法：DDL、DQL](https://www.cnblogs.com/saowei/p/15916490.html)
- [Hive 常用函数](https://www.cnblogs.com/saowei/p/15916717.html)
- [Hive 高级函数----开窗函数](https://www.cnblogs.com/saowei/p/15921127.html)
- [Hive 行列转换](https://www.cnblogs.com/saowei/p/15928022.html)
- [Hive结合shell脚本企业实战用法](https://www.cnblogs.com/saowei/p/15928996.html)
- [Hive调优](https://www.cnblogs.com/saowei/p/15934146.html)
- [Hive数仓建模（架构设计）](https://www.cnblogs.com/saowei/p/15934163.html)

Hive将表划分为分区(partition)表和分桶(bucket)表。 分区表在加载数据的时候可以指定加载某一部分数据，并不是全量的数据，可以让数据的部分查询变得更快。分桶表通常是在原始数据中加入一些额外的结构，这些结构可以用于高效的查询，例如，基于ID的分桶可以使得用户的查询非常的块。

分区表与分桶表是可以一起使用的。

# 1 Hive分区表

## 1.1 Hive分区表的概念？

Hive分区是将数据表的某一个字段或多个字段进行统一归类，而后存储在在hdfs上的不同文件夹中。当查询过程中指定了分区条件时，只将该分区对应的目录作为Input，从而减少MapReduce的输入数据，提高查询效率，这也是数仓优化的一个列，也就是分区裁剪。分区表又分为静态分区表和动态分区表两种。这也是数仓性能优化的一个常用点，也就是分区裁剪。

分区表又分为静态分区表和动态分区表两种。

分区的概念提供了一种将Hive表数据分离为多个文件/目录的方法。**不同分区对应着不同的文件夹，同一分区的数据存储在同一个文件夹下**。只需要根据分区值找到对应的文件夹，扫描本分区下的文件即可，避免全表数据扫描。

**Hive 中分区表的意思是按照表的某一列列名(1个字段)或某几列列名(多个字段)作为类似文件夹的形式来隔离分开存放数据，以便提高检索效率和管理效率。**

### 1.1.1 分区表注意事项

1. 分区表不是建表的必要语法规则，是一种优化手段表，可选；

2. 分区**字段不能是表中已有的字段**，不能重复；

3. 分区字段是**虚拟字段**，其数据并不存储在底层的文件中；

4. 分区字段值的确定来自于用户价值数据手动指定（**静态分区**）或者根据查询结果位置自动推断（**动态分区**）

5. Hive**支持多重分区**，也就是说在分区的基础上继续分区，划分更加细粒度

### 1.1.2 分区表物理存储结构

分区表表在hdfs上作为一个文件夹存在，添加分区之后就可以在hdfs文件系统当中看到表下面多了一个文件夹

```bash
0: jdbc:hive2://node03:10000> dfs -ls /user/hive/warehouse/myhive1.db/score;
+----------------------------------------------------+--+
|                     DFS Output                     |
+----------------------------------------------------+--+
| Found 4 items                                      |
| drwxr-xr-x   - hadoop supergroup          0 2020-06-07 15:57 /user/hive/warehouse/myhive1.db/score/month=201803 |
| drwxr-xr-x   - hadoop supergroup          0 2020-06-07 15:57 /user/hive/warehouse/myhive1.db/score/month=201804 |
| drwxr-xr-x   - hadoop supergroup          0 2020-06-07 15:57 /user/hive/warehouse/myhive1.db/score/month=201805 |
| drwxr-xr-x   - hadoop supergroup          0 2020-06-07 15:53 /user/hive/warehouse/myhive1.db/score/month=201806 |
```

### 1.1.3 分区表使用场景

实际工作中分区表常常被运用于按照某一维度进行统计分析的场景下，数据被按照某一个日期、年月日等等，将一个大的文件切分成一个个小文件，分而治之，这样处理起来性能就会有显著提升。

**分区表建表语法：**

```sql
CREATE TABLE table_name (column1 data_type, column2 data_type) PARTITIONED BY (partition1 data_type, partition2 data_type,….);
```

需要注意：**分区字段不能是表中已经存在的字段**，因为分区字段最终也会以虚拟字段的形式显示在表结构上。

## 1.2 分区操作

### 1.2.1 分区表创建语法

分区表 hive sql 创建语句如下：

```sql
create table login_logs(l_id string, l_loginName string, l_date string) partitioned by (date string) row format delimited fields terminated by  '\t';
```

**分区表在创建时关键字是：partitioned by**

括号里跟上分区字段名及其字段类型，这样在创建表时就指明要创建的是一个分区表，上述创建语句说明要创建一个分区字段为：date，且类型为 string 类型的分区表，表名是 login_logs。

### 1.2.2 多分区表创建方式

多分区表 hive sql 创建语句如下：

```sql
create table login_logs(l_id string, l_loginName string, l_date string) partitioned by (year string, month string) row format delimited fields terminated by  '\t';
```

与1.2中分区表创建语句的区别在于关键字 partitioned by 后面的参数是指定了多个分区字段，每个分区字段都需要指定字段类型，多字段之间以逗号分隔。

**例如：** dws_test_001_daily_df 表有2个分区字段，分区字段分别是：dt，hotel。

**hdfs多分区（文件夹）信息查询：一级分区（文件夹）**

```bash
hive>dfs -ls /user/hive/warehouse/dws.db/dws_test_001_daily_df;
```

### 1.2.3 修改分区

**命令：**

```javascript
ALTER TABLE table_name PARTITION (dt='2021-08-08') SET LOCATION "new location";
ALTER TABLE table_name PARTITION (dt='2021-08-08') RENAME TO PARTITION (dt='20210808');
```

#### 1.2.4 删除分区

**命令：**

```javascript
ALTER TABLE tmp_table DROP IF EXISTS PARTITION (dt="2021-12-01", hotel= "AAA");
```



## 1.4 静态分区表是什么？

静态分区表：所谓的静态分区表指的就是，我们在创建表的时候，就已经给该表中的数据定义好了数据类型，在进行加载数据的时候，我们已经知道该数据属于什么类型，并且直接加载到该分区内就可以了。

语法如下：

```go
load data [local] inpath ' ' into table tablename partition(分区字段='分区值'...);
```

Local表示数据是位于本地文件系统还是HDFS文件系统。

### 1.4.1 静态分区表案例

```sql
-- 创建分区表
hive (myhive)> create table score(s_id string, c_id string, s_score int) partitioned by (month string) row format delimited fields terminated by '\t';
-- 创建一个表带多个分区
hive (myhive)> create table score2 (s_id string,c_id string, s_score int) partitioned by (year string, month string, day string) row format delimited fields terminated by '\t';
-- 加载数据到静态分区表当中去
hive (myhive)>load data local inpath '/kkb/install/hivedatas/score.csv' into table score partition (month='201806');
-- 加载数据到多分区静态分区表当中去
hive (myhive)>load data local inpath '/kkb/install/hivedatas/score.csv' into table score partition (month='201806');
-- 加载数据到多分区静态分区表当中去
hive (myhive)> load data local inpath '/kkb/install/hivedatas/score.csv' into table score2 partition(year='2018', month='06', day='01');
-- 查看分区
show  partitions  score;
+---------------+--+
|   partition   |
+---------------+--+
| month=201803  |
| month=201804  |
| month=201805  |
| month=201806  |
+---------------+--+
-- 创建分区（怎么添加hive分区）
alter table score add partition(month='201805');
alter table score add partition(month='201804') partition(month = '201803');
-- 删除分区（怎么删除hive分区）
alter table score drop partition(month = '201806');
```

特殊说明：同内部表和外部表一致，如果该分区表为外部表，则分区对应的HDFS目录数据不会被删除。

### 1.4.2 分区表练习一

> 需求描述
>
> - 现在有一个文件score.csv文件，里面有三个字段，分别是s_id string, c_id string, s_score int
> - 字段都是使用 \t进行分割
> - 存放在集群的这个目录下/scoredatas/day=20180607，这个文件每天都会生成，存放到对应的日期文件夹下面去
> - 文件别人也需要公用，不能移动
> - 请创建hive对应的表，并将数据加载到表中，进行数据统计分析，且删除表之后，数据不能删除

```sql
-- 本地上传数据到hdfs
cd /opt/module/hive-1.1.0-cdh5.14.2/data/test
hdfs dfs -mkdir -p /scoredatas/day=20180607
hdfs dfs -put score.csv /scoredatas/day=20180607/
-- 创建外部分区表，并指定文件数据存放目录
create external table score4(s_id string, c_id string, s_score int) partitioned by (day string) row format delimited fields terminated by '\t' location '/scoredatas';
-- 进行hive的数据表修复，说白了就是建立我们表与我们数据文件之间的一个关系映射(),修复成功之后即可看到数据已经全部加载到表当中去了
msck repair table score4;
```

### 1.4.3 分区操作

- 一次添加多个分区

```sql
ALTER TABLE page_view ADD PARTITION (dt='2008-08-08', country='us') location '/path/to/us/part080808' PARTITION (dt='2008-08-09', country='us') location '/path/to/us/part080809'; 
```

- 删除分区

```sql
ALTER TABLE login DROP IF EXISTS PARTITION (dt='2008-08-08');
 
ALTER TABLE page_view DROP IF EXISTS PARTITION (dt='2008-08-08', country='us');
```

- 修改分区

```sql
ALTER TABLE table_name PARTITION (dt='2008-08-08') SET LOCATION "new location";
ALTER TABLE table_name PARTITION (dt='2008-08-08') RENAME TO PARTITION (dt='20080808');
```

- 表的重命名

```sql
ALTER TABLE table_name RENAME TO new_table_name
```

## 1.5 动态分区表是什么？

动态分区表：所谓的动态分区表，其实建表方式跟静态分区表没有区别，最主要的区别是在载入数据的时候，静态分区表我们载入数据之前必须保证该分区存在，并且已经明确知道载入的数据的类型，知道要将数据加载到那个分区当中去，而动态分区表，在载入的时候，我们事先并不知道该条数据属于哪一类，而是需要hive自己去判断该数据属于哪一类，并将该条数据加载到对应的目录中去。

建表语句跟静态分区表的建表语句相同，这里不再赘述，主要来看看数据的加载：

对于动态分区表数据的加载，我们需要先开启hive的非严格模式，并且通过insert的方式进行加载数据。

所谓**动态分区**指的是分区的字段值是基于查询结果自动推断出来的。核心语法就是insert+select。

启用hive动态分区，需要在hive会话中设置两个参数：

```bash
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
```

第一个参数表示开启动态分区功能，第二个参数指定动态分区的模式。分为nonstick非严格模式和strict严格模式。strict严格模式要求至少有一个分区为静态分区。

### 1.5.1 动态态分区表案例（如何实现动态分区）

这里针对hive动态分区的实现进行说明，实例内容是在一个普通表里面存在订单时间的字段，将普通数据表的数据按照订单时间作为分区字段，动态分区到分区表中。

```sql
-- 创建分区表，分别创建普通表，和分区表
-- 创建普通表
create table t_order(
                        order_number string,
                        order_price  double,
                        order_time   string
)row format delimited fields terminated by '\t';
-- 创建目标分区表
create table order_dynamic_partition(
                                        order_number string,
                                        order_price  double
)partitioned BY(order_time string)
row format delimited fields terminated by '\t';
-- 导入准备好的数据
cd /opt/module/hive-1.1.0-cdh5.14.2/data/test
vim order_partition.txt
10001   100 2019-03-02 
10002   200 2019-03-02
10003   300 2019-03-02
10004   400 2019-03-03
10005   500 2019-03-03
10006   600 2019-03-03
10007   700 2019-03-04
10008   800 2019-03-04
10009   900 2019-03-04
-- 普通表t_order load加载数据
load data local inpath '/opt/module/hive-1.1.0-cdh5.14.2/data/test/order_partition.txt' overwrite into table t_order;
-- 动态分区的实现， 将普通表的数据按照order_time字段作为分区字段，动态加载数据到分区表中
hive> set hive.exec.dynamic.partition=true;             -- 开启动态分区功能
hive> set hive.exec.dynamic.partition.mode=nonstrict;   -- 设置hive为非严格模式
hive> insert into table order_dynamic_partition partition(order_time) select order_number, order_price, order_time from t_order;
-- 查看分区
show partitions order_dynamic_partition;
+------------------------+--+
|       partition        |
+------------------------+--+
| order_time=2019-03-02  |
| order_time=2019-03-03  |
| order_time=2019-03-04  |
+------------------------+--+
```

开启开启动态分区的条件和注意事项：

1、必须先开启动态分区模式为非严格模式

2、在指定分区的时候，并没有指定具体分区的值，而只是指定的分区的字段

3、partition中的字段其实是作为插入目标表中的一个字段，所以在从另外一张表select的时候必须查询字段中包含索要分区的这个字段。

## 1.6 Hive 分区表 & 数据加载方式

### 1.6.1 本地上传至hdfs

命令：

```javascript
hdfs dfs -put [文件名] [hdfs绝对路径]
```

### 1.6.2 本地导入

**命令：**

```javascript
load data local inpath [本地文件路径] into table [表名];
```

**示例：**

```javascript
load data local inpath '/user/xiaomin.liu/hive_testdata/student.csv' into table student;
```

**含义：**

将 /user/xiaomin.liu/hive_testdata 目录下 student.csv 文件的内容加载至 hdfs 中的 student 表中。

**注意：**

**1. student.csv 文件内容中：数据类型、数据列数、列数之间的分隔符要与 hdfs 中 student 表定义的一一对应。**

**2. 使用本地导入的方式加载数据至 hive，使用的是复制操作，即当本地路径下的文件被加载至 hive 后，该本地路径下的文件依然存在，不会消失。**

#### 1.6.3 hdfs导入

**命令：**

```javascript
load data inpath [文件在hdfs中的路径] into table [表名];
```

**示例：**

```javascript
load data inpath '/user/warehouse/test/student.txt' into table student;
```

**含义：**

将 hdfs 路径为 /user/warehouse/test 下的 student.txt 文件的数据加载至 hdfs 中的 student 表中。

**注意：**

**使用 hdfs 导入数据至 hive，使用的是剪切操作，即原 hdfs 路径下的文件在被导入至 hive 后，原 hdfs 路径下的文件将不存在了。**

### 1.6.4 覆盖导入

**本地导入命令：**

```javascript
load data local inpath [本地文件路径] overwrite into table [表名];
```

**hdfs导入命令：**

```javascript
load data inpath [hdfs文件路径] overwrite into table [表名];
```

**示例：** **本地导入命令：**

```javascript
load data local inpath '/user/xiaomin.liu/hive_testdata/student.csv' overwrite into table student;
```

**hdfs导入命令：**

```javascript
load data inpath '/user/warehouse/test/student.txt' overwrite into table student;
```

**含义：**

**同1.2中的本地导入和1.3中的 hdfs 导入类似，只是新增的关键字 overwrite 会将表中已有的数据进行覆盖操作。**

### 1.6.5 查询导入

**命令：**

```javascript
create table [目标table_name] as select * from [已存在table_name];
```

**示例：**

```javascript
create table sub_student as select * from student;
```

**含义：**将表 student 的结构与数据复制一份给到表 sub_student。

### 1.6.6 insert导入

**追加模式命令：**

```javascript
insert into table [表名] select * from [已存在table_name];
```

**示例：**

```javascript
insert into table sub_student select * from student;
```

**含义：**将表 student 的数据以 append 追加的方式写入表 sub_student 中。

**覆盖模式命令：**

```javascript
insert overwrite table [表名] select * from [已存在table_name];
```

**示例：**

```javascript
insert overwrite table sub_student select * from student;
```

**含义：**将表 student 的数据以覆盖的方式写入表 sub_student 中。

# 2 Hive分桶表

## 2.1 Hive分桶表概念？

Hive分桶是相对分区进行更细粒度的划分。是将整个数据内容按照某列取hash值，对桶的个数取模的方式决定该条记录存放在哪个桶当中；具有相同hash值的数据进入到同一个文件中。 如要安装name属性分为3个桶，就是对name属性值的hash值对3取摸，按照取模结果对数据分桶。如取模结果为0的数据记录存放到一个文件，取模为1的数据存放到一个文件，取模为2的数据存放到一个文件。

## 2.2 创建分桶表

在创建分桶表之前要执以下的命令，开启对分桶表的支持以及reduce个数

```sql
set hive.enforce.bucketing=true;
# 设置与桶相同的reduce个数（默认只有一个reduce）
set mapreduce.job.reduces=4;COPY
```

创建分桶表

```sql
create table myhive1.user_buckets_demo(id int, name string)
clustered by(id) 
into 4 buckets 
row format delimited fields terminated by '\t';COPY
```

如何向分桶表中导入数据

向分桶表中导入数据，不可以直接加载，需要先导入普通表，再导入分桶表中，这种和动态分区类似。

```sql
# 创建普通表
create table user_demo(id int, name string)
row format delimited fields terminated by '\t';
# 准备数据文件 buckets.txt
cd /opt/module/hive-1.1.0-cdh5.14.2/data/test
vim user_bucket.txt
1   anzhulababy1
2   anzhulababy2
3   anzhulababy3
4   anzhulababy4
5   anzhulababy5
6   anzhulababy6
7   anzhulababy7
8   anzhulababy8
9   anzhulababy9
10  anzhulababy10
# 向普通标中导入数据
load data local inpath '/opt/module/hive-1.1.0-cdh5.14.2/data/test/user_bucket.txt'  overwrite into table user_demo; 
# 查看数据
select * from user_demo;
+---------------+-----------------+--+
| user_demo.id  | user_demo.name  |
+---------------+-----------------+--+
| 1             | anzhulababy1    |
| 2             | anzhulababy2    |
| 3             | anzhulababy3    |
| 4             | anzhulababy4    |
| 5             | anzhulababy5    |
| 6             | anzhulababy6    |
| 7             | anzhulababy7    |
| 8             | anzhulababy8    |
| 9             | anzhulababy9    |
| 10            | anzhulababy10   |
+---------------+-----------------+--+
# 加载数据到桶表user_buckets_demo中
insert into table user_buckets_demo select * from user_demo;COPY
```

## 2.3 分桶表物理存储结构

分桶表表在hdfs上作为一个文件存在。

```sql
0: jdbc:hive2://node03:10000> dfs -ls /user/hive/warehouse/myhive1.db/user_buckets_demo;
+----------------------------------------------------+--+
|                     DFS Output                     |
+----------------------------------------------------+--+
| Found 4 items                                      |
| -rwxr-xr-x   3 hadoop supergroup         30 2020-06-08 13:30 /user/hive/warehouse/myhive1.db/user_buckets_demo/000000_0 |
| -rwxr-xr-x   3 hadoop supergroup         45 2020-06-08 13:30 /user/hive/warehouse/myhive1.db/user_buckets_demo/000001_0 |
| -rwxr-xr-x   3 hadoop supergroup         47 2020-06-08 13:30 /user/hive/warehouse/myhive1.db/user_buckets_demo/000002_0 |
| -rwxr-xr-x   3 hadoop supergroup         30 2020-06-08 13:30 /user/hive/warehouse/myhive1.db/user_buckets_demo/000003_0 |
+----------------------------------------------------+--+
```

## 2.4 分桶表使用场景

- 取样sampling更高效。没有分桶的话需要扫描整个数据集。
- 提升某些查询操作效率，例如map side join

## 2.5 如何抽样查询桶表的数据

tablesample抽样语句语法：tablesample(bucket x out of y)

- x表示从第几个桶开始取数据
- y与进行采样的桶数的个数、每个采样桶的采样比例有关

```sql
select * from user_buckets_demo tablesample(bucket 1 out of 2);
-- 需要采样的总桶数=4/2=2个
-- 先从第1个桶中取出数据
-- 1+2=3，再从第3个桶中取出数据
```

# 3 内部表

- [Hive 内部表与外部表](https://cloud.tencent.com/developer/article/1481751)
- [六、Hive中的内部表、外部表、分区表和分桶表](https://cloud.tencent.com/developer/article/2075653)

托管表(`Managed TABLE`)也称为内部表(`Internal TABLE`)。这是Hive中的默认表。当我们在Hive中创建一个表，没有指定为外部表时，默认情况下我们创建的是一个内部表。如果我们创建一个内部表，那么表将在`HDFS`中的特定位置创建。默认情况下，表数据将在`HDFS`的`/usr/hive/warehouse`目录中创建。如果我们删除了一个内部表，那么这个表的表数据和元数据都将从`HDFS`中删除。

## 3.1 创建表

我们可以用下面的语句在Hive里面创建一个内部表：

```sql
CREATE  TABLE IF NOT EXISTS tb_station_coordinate(
  station string,
  lon string,
  lat string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
```

我们已经成功创建了表并使用如下命令检查表的详细信息：

```sql
hive> describe formatted tb_station_coordinate;
OK
# col_name            	data_type           	comment             

station             	string              	                    
lon                 	string              	                    
lat                 	string              	                    

# Detailed Table Information	 	 
Database:           	default             	 
Owner:              	xiaosi              	 
CreateTime:         	Tue Dec 12 17:42:09 CST 2017	 
LastAccessTime:     	UNKNOWN             	 
Retention:          	0                   	 
Location:           	hdfs://localhost:9000/user/hive/warehouse/tb_station_coordinate	 
Table Type:         	MANAGED_TABLE       	 
Table Parameters:	 	 
	COLUMN_STATS_ACCURATE	{\"BASIC_STATS\":\"true\"}
	numFiles            	0                   
	numRows             	0                   
	rawDataSize         	0                   
	totalSize           	0                   
	transient_lastDdlTime	1513071729          

# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
InputFormat:        	org.apache.hadoop.mapred.TextInputFormat	 
OutputFormat:       	org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat	 
Compressed:         	No                  	 
Num Buckets:        	-1                  	 
Bucket Columns:     	[]                  	 
Sort Columns:       	[]                  	 
Storage Desc Params:	 	 
	field.delim         	,                   
	serialization.format	,                   
Time taken: 0.16 seconds, Fetched: 33 row(s)
```

从上面我们可以看到表的类型`Table Type`为`MANAGED_TABLE`，即我们创建了一个托管表(内部表)。

### 3.2 导入数据

我们使用如下命令将一个样本数据集导入到表中：

```sql
hive> load data local inpath '/home/xiaosi/station_coordinate.txt' overwrite into table tb_station_coordinate;
Loading data to table default.tb_station_coordinate
OK
Time taken: 2.418 seconds
```

如果我们在`HDFS`的目录`/user/hive/warehouse/tb_station_coordinate`查看，我们可以得到表中的内容：

```sql
xiaosi@yoona:~$ hadoop fs -ls  /user/hive/warehouse/tb_station_coordinate
Found 1 items
-rwxr-xr-x   1 xiaosi supergroup        374 2017-12-12 17:50 /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
xiaosi@yoona:~$
xiaosi@yoona:~$
xiaosi@yoona:~$ hadoop fs -text  /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
桂林北站,110.302159,25.329024
杭州东站,120.213116,30.290998
山海关站,119.767555,40.000793
武昌站,114.317576,30.528401
北京南站,116.378875,39.865052
...
```

\> `/home/xiaosi/station_coordinate.txt`是本地文件系统路径。从上面的输出我们可以看到数据是从本地的这个路径复制到`HDFS`上的`/user/hive/warehouse/tb_station_coordinate/`目录下。 为什么会自动复制到`HDFS`这个目录下呢？这个是由`Hive`的配置文件设置的。在`Hive`的`${HIVE_HOME}/conf/hive-site.xml`配置文件中指定，`hive.metastore.warehouse.dir`属性指向的就是`Hive`表数据存放的路径(在这配置的是`/user/hive/warehouse/`)。`Hive`每创建一个表都会在`hive.metastore.warehouse.dir`指向的目录下以表名创建一个文件夹，所有属于这个表的数据都存放在这个文件夹里面`/user/hive/warehouse/tb_station_coordinate`。

## 3.3 删除表

现在让我们使用如下命令删除上面创建的表:

```javascript
hive> drop table tb_station_coordinate;
Moved: 'hdfs://localhost:9000/user/hive/warehouse/tb_station_coordinate' to trash at: hdfs://localhost:9000/user/xiaosi/.Trash/Current
OK
Time taken: 1.327 seconds
```

从上面的输出我们可以得知，原来属于`tb_station_coordinate`表的数据被移到`hdfs://localhost:9000/user/xiaosi/.Trash/Current`文件夹中(如果你的Hadoop没有采用回收站机制，那么删除操作将会把属于该表的所有数据全部删除)(回收站机制请参阅:[Hadoop Trash回收站使用指南](http://smartying.club/2017/12/07/Hadoop/Hadoop Trash回收站使用指南/))。

如果我们在`HDFS`的目录`/user/hive/warehouse/tb_station_coordinate`查看：

```javascript
xiaosi@yoona:~$ hadoop fs -ls  /user/hive/warehouse/tb_station_coordinate
ls: `/user/hive/warehouse/tb_station_coordinate': No such file or directory
```

你可以看到输出为`No such file or directory`，因为表及其内容都从HDFS从删除了。

# 4 外部表

当数据在Hive之外使用时，创建外部表(`EXTERNAL TABLE`)来在外部使用。无论何时我们想要删除表的元数据，并且想保留表中的数据，我们使用外部表。外部表只删除表的`schema`。

## 4.1 外部普通表

我们使用如下命令创建一个外部表：

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS tb_station_coordinate(
  station string,
  lon string,
  lat string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
```

我们现在已经成功创建了外部表。我们使用如下命令检查关于表的细节：

```sql
hive> describe formatted tb_station_coordinate;
OK
# col_name            	data_type           	comment             

station             	string              	                    
lon                 	string              	                    
lat                 	string              	                    

# Detailed Table Information	 	 
Database:           	default             	 
Owner:              	xiaosi              	 
CreateTime:         	Tue Dec 12 18:16:13 CST 2017	 
LastAccessTime:     	UNKNOWN             	 
Retention:          	0                   	 
Location:           	hdfs://localhost:9000/user/hive/warehouse/tb_station_coordinate	 
Table Type:         	EXTERNAL_TABLE      	 
Table Parameters:	 	 
	COLUMN_STATS_ACCURATE	{\"BASIC_STATS\":\"true\"}
	EXTERNAL            	TRUE                
	numFiles            	0                   
	numRows             	0                   
	rawDataSize         	0                   
	totalSize           	0                   
	transient_lastDdlTime	1513073773          

# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
InputFormat:        	org.apache.hadoop.mapred.TextInputFormat	 
OutputFormat:       	org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat	 
Compressed:         	No                  	 
Num Buckets:        	-1                  	 
Bucket Columns:     	[]                  	 
Sort Columns:       	[]                  	 
Storage Desc Params:	 	 
	field.delim         	,                   
	serialization.format	,                   
Time taken: 0.132 seconds, Fetched: 34 row(s)
```

从上面我们可以看到表的类型`Table Type`为`EXTERNAL_TABLE`，即我们创建了一个外部表。

## 4.2 导入数据

我们使用如下命令将一个样本数据集导入到表中：

```sql
hive> load data local inpath '/home/xiaosi/station_coordinate.txt' overwrite into table tb_station_coordinate;
Loading data to table default.tb_station_coordinate
OK
Time taken: 2.418 seconds
```

如果我们在`HDFS`的目录`/user/hive/warehouse/tb_station_coordinate`查看，我们可以得到表中的内容：

```sql
xiaosi@yoona:~$ hadoop fs -ls  /user/hive/warehouse/tb_station_coordinate
Found 1 items
-rwxr-xr-x   1 xiaosi supergroup        374 2017-12-12 18:19 /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
xiaosi@yoona:~$
xiaosi@yoona:~$
xiaosi@yoona:~$ hadoop fs -text  /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
桂林北站,110.302159,25.329024
杭州东站,120.213116,30.290998
山海关站,119.767555,40.000793
武昌站,114.317576,30.528401
...
```

## 4.3 删除表

现在让我们使用如下命令删除上面创建的表:

```sql
hive> drop table tb_station_coordinate;
OK
Time taken: 0.174 seconds
hive>
```

我们的Hadoop已经开启了回收站机制，但是删除操作并没有将数据进行删除，不像删除内部表一样，输出`Moved: 'hdfs://localhost:9000/user/hive/warehouse/tb_station_coordinate' to trash at: hdfs://localhost:9000/user/xiaosi/.Trash/Current`(回收站机制请参阅:[Hadoop Trash回收站使用指南](http://smartying.club/2017/12/07/Hadoop/Hadoop Trash回收站使用指南/))。为了验证我们真的没有删除数据，我们在HDFS目录下查看数据:

```sql
xiaosi@yoona:~$ hadoop fs -ls  /user/hive/warehouse/tb_station_coordinate
Found 1 items
-rwxr-xr-x   1 xiaosi supergroup        374 2017-12-12 18:19 /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
xiaosi@yoona:~$
xiaosi@yoona:~$ hadoop fs -text  /user/hive/warehouse/tb_station_coordinate/station_coordinate.txt
桂林北站,110.302159,25.329024
杭州东站,120.213116,30.290998
山海关站,119.767555,40.000793
武昌站,114.317576,30.528401
北京南站,116.378875,39.865052
...
```

你可以看到表中的数据仍然在HDFS中。所以我们得知如果我们创建一个外部表，在删除表之后，只有与表相关的元数据被删除，而不会删除表的内容。

## 4.4 创建表指定外部目录

只有当你的数据在`/user/hive/warehouse`目录中时，上述方法才能有效。但是，如果你的数据在另一个位置，如果你删除该表，数据也将被删除。所以在这种情况下，你需要在创建表时设置数据的外部位置，如下所示：

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS tb_station_coordinate(
  station string,
  lon string,
  lat string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/xiaosi/test/coordinate/';
```

>  备注: 你也可以通过在创建表时设置[数据存储](https://cloud.tencent.com/product/cdcs?from=10680)位置来创建一个内部表。但是，如果删除表，数据将被删除。 如果你想要创建外部表，需要在创建表的时候加上 EXTERNAL 关键字，同时指定外部表存放数据的路径(例如2.4所示)，也可以不指定外部表的存放路径(例如2.3所示)，这样Hive将在HDFS上的/user/hive/warehouse/目录下以外部表的表名创建一个文件夹，并将属于这个表的数据存放在这里。

# 5 内部表和外部表使用场景

## 5.1 内部表

- 数据是临时的
- 希望使用`Hive`来管理表和数据的生命周期
- 删除后不想要数据

## 5.2 外部表

- 这些数据也在`Hive`之外使用。
- `Hive`不管理数据和权限设置以及目录等，需要你有另一个程序或过程来做这些事情
- 不是基于现有表(AS SELECT)来创建的表
- 可以创建表并使用相同的模式并指向数据的位置