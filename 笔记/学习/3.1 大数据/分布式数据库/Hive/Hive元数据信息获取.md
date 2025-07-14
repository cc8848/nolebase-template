- [如何获取Hive库表字段分区信息](https://blog.csdn.net/baidu_32377671/article/details/111728347)

# 1 获取Hive的所有数据库信息

**如果注释字段中文乱码，则需要修改字符集（正常情况都需要修改）。**

```sql
SELECT
	`NAME` NAME,
	ANY_VALUE ( `DESC` ) COMMENT,
	ANY_VALUE ( `DB_LOCATION_URI` ) path 
FROM
	`DBS` 
GROUP BY
	`NAME
```

# 2 根据数据库名获取数据表信息

**`path`字段对应着Hive表在Hdfs的路径，在做数据同步的时候很重要（比如使用`dataX`进行数据同步，当然手写JSON也是很累的，但是还好有datax-web开源项目）,`type`字段可以看出表的储存格式(目前已知Hive可以直接加载Hdfs目录中的`text格式`和`parquet格式`文件)。**

```sql
SELECT
	tb.TBL_NAME NAME,
	tp.PARAM_VALUE COMMENT,
	sds.INPUT_FORMAT type,
	sds.LOCATION path 
FROM
	DBS db
	LEFT JOIN TBLS tb ON tb.DB_ID = db.DB_ID
	LEFT JOIN TABLE_PARAMS tp ON tp.TBL_ID = tb.TBL_ID 
	AND tp.PARAM_KEY = 'comment'
	LEFT JOIN SDS sds ON sds.SD_ID = tb.SD_ID 
WHERE
	db.`NAME` = 'test'
ORDER BY
	tb.TBL_NAME
```

# 3 根据数据库名、数据表名获取表字段信息

**此处查询结果只会展示表的字段信息（分区字段不会在该查询语句的结果中显示）。**

```sql
SELECT
	col.COLUMN_NAME NAME,
	col.TYPE_NAME type,
	col.`COMMENT` COMMENT,
	col.INTEGER_IDX position 
FROM
	DBS db
	LEFT JOIN TBLS tb ON tb.DB_ID = db.DB_ID
	LEFT JOIN SDS sds ON sds.SD_ID = tb.SD_ID
	LEFT JOIN COLUMNS_V2 col ON sds.CD_ID = col.CD_ID 
WHERE
	db.NAME = 'test' 
	AND tb.TBL_NAME = 'person'
ORDER BY
	col.INTEGER_IDX
```

# 4 根据数据库名、数据表名获取表分区信息

**至于分区字段为什么单独拿出来不合到表字段里，是因为每个分区对应着一个Hdfs目录。分区有目录，字段没有啊怎么能放到一起。（在使用datax同步分区数据时会将分区数据同步到分区表指定的分区目录中）。**

## 4.1 查看分区表分区字段信息

**创建一个分区表用年、月、日三级目录进行分区：**

```sql
create table if not exists person_partition(
    name string comment '姓名',
    age  int comment '年龄'
) comment '测试表'
partitioned by (
    year string comment '年',
    month string comment '月',
    day string comment '日'
)
stored as parquet;
```



```sql
SELECT
	pk.PKEY_NAME NAME,
	pk.PKEY_TYPE type,
	pk.PKEY_COMMENT COMMENT,
	pk.INTEGER_IDX position 
FROM
	DBS db
	LEFT JOIN TBLS tb ON tb.DB_ID = db.DB_ID
	LEFT JOIN PARTITION_KEYS pk ON pk.TBL_ID = tb.TBL_ID 
WHERE
	db.`NAME` = 'test' 
	AND tb.TBL_NAME = 'person_partition' 
ORDER BY
	pk.INTEGER_IDX
```

## 4.2 查看分区表的分区信息

**当前表中还没有分区，先来加几个分区：**

- 添加分区【同时会在Hdfs中创建目录】:`alter table [table] add partition(col,..);`
- 删除分区【同时会在Hdfs中删除目录】:`alter table [table] drop partition(col,..);`

```sql
alter table person_partition add if not exists partition(year=2015,month=12,day=26);
alter table person_partition add if not exists partition(year=2016,month=12,day=26);
alter table person_partition add if not exists partition(year=2017,month=12,day=26);
alter table person_partition add if not exists partition(year=2018,month=12,day=26);
alter table person_partition add if not exists partition(year=2019,month=12,day=26);
alter table person_partition add if not exists partition(year=2020,month=12,day=26);
```

**接下来查询下分区表的分区信息，看看每个分区对应的Hdfs路径（使用datax数据同步时将会使用到）：**

```sql
SELECT
	db.`NAME` dbName,
	tb.TBL_NAME tbName,
	part.PART_NAME partName,
	sds.LOCATION path 
FROM
	`PARTITIONS` part
	LEFT JOIN TBLS tb ON tb.TBL_ID = part.TBL_ID
	LEFT JOIN DBS db ON db.DB_ID = tb.DB_ID
	LEFT JOIN SDS sds ON sds.SD_ID = part.SD_ID 
WHERE
	db.`NAME` = 'test' 
	AND tb.TBL_NAME = 'person_partition'
```

# 附录

## A.1 Hive元数据中文乱码

hive配置表的[元数据](https://so.csdn.net/so/search?q=元数据&spm=1001.2101.3001.7020)放到mysql中存储，mysql中默认表被创建的时候用的是默认的字符集(latin1)，所以会出现中文乱码。

解决方案：

（1）在Hive元数据存储的Mysql数据库（MetaStore）中，执行以下SQL：

```sql
#修改字段注释字符集
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
#修改表注释字符集
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(20000) character set utf8;
#修改分区参数，支持分区建用中文表示
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(20000) character set utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(20000) character set utf8;
#修改索引名注释，支持中文表示
alter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
#修改视图，支持视图中文
ALTER TABLE TBLS modify COLUMN VIEW_EXPANDED_TEXT mediumtext CHARACTER SET utf8;
ALTER TABLE TBLS modify COLUMN VIEW_ORIGINAL_TEXT mediumtext CHARACTER SET utf8;
```

（2）修改hive-site.xml中Hive读取元数据的编码（注意原先是否配置过）

```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
</property>
```

注意的是，之前创建的表的元数据已经损坏了，所以无法恢复中文乱码，只能重新创建表