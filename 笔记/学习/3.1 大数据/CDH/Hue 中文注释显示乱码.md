- [Hue 中文注释显示乱码_csdn-延的博客-CSDN博客_hue页面的字符编码](https://blog.csdn.net/m0_37125796/article/details/104829770)

 1. 进入mysql ，查看hive 数据库当前编码

```sql
show create database hive ;
```

2. 如果是utf8 则执行下面sql将 hive 数据库默认编码改成 latin1

```sql
alter database hive default character set latin1 ;
```

3. 执行下面sql ，修改 表/字段/分区/索引 等部分的注释字符集

```sql
use hive; 
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_PARAMS  modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_KEYS  modify column PKEY_COMMENT varchar(4000) character set utf8;
alter table  INDEX_PARAMS  modify column PARAM_VALUE  varchar(4000) character set utf8;
```

额外补充:

**为什么数据库字符编码要用latin1(iso-8859-1),而不是utf8呢?** 

​      因为在mysql innodb引擎中，索引的长度最大字节数为767。 utf8 数字、英文1字节、中文3字节，gbk，数字、英文、中文各2字节，latin1，数字、英文、中文各1字节，所以hive[元数据](https://so.csdn.net/so/search?q=元数据&spm=1001.2101.3001.7020)库为了获取更长的索引长度就采取了latin1字符编码。 [其它](https://www.cnblogs.com/kerrycode/p/9680881.html)