- [数据仓库之【用户行为数仓】06：【dwd层】明细数据层：清洗ods层之后的数据_做一个有趣的人Zz的博客-CSDN博客_dwd层数据清洗](https://blog.csdn.net/weixin_40612128/article/details/123386913)

# 一、dwd层分析

下面我们来创建dwd层：

针对ods层表中的数据进行清洗，参考数据清洗规则，按照实际情况对数据进行清洗。

```
注意：如果清洗规则使用SQL可以实现，那么就使用SQL实现数据清洗，如果清洗的规则使用SQL实现起来非常麻烦，或者使用SQL压根无法实现，此时就可以考虑需要使用MapReduce代码或者Spark代码对数据进行清洗了。
```

由于我们这里采集的数据还是比较规整的，可以使用SQL实现，所以我们就直接使用SQL实现数据清洗了。

# 二、创建dwd层数据库

在hive中创建数据库dwd_mall

```
create database dwd_mall;
show databases;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/259a38a9dea844649b04858ac3557004.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_19,color_FFFFFF,t_70,g_se,x_16)

# 三、创建dwd层的表

注意：

```
1、原始json数据中的用户id字段名称为uid,但是在商品订单数据中用户id字段名称为user_id,这块需要注意一下，在实际工作中会有这种情况，客户端和服务端数据的个别字段名称不一致，所以我们在使用的时候最好是统一一下，后期使用起来比较方便，所以在这里我会通过uid解析数据，解析之后，给字段起别名为user_id。

2、hive中的timestamp只能解析yyyy-MM-dd HH:MM:SS格式的数据，所以针对里面的acttime字段我们使用bigint类型。

3、为了考虑到SQL重跑的情况，在使用insert into table 的时候最好改为insert overwrite table ,否则SQL重复执行的时候会重复写入数据。
```

## 1、dwd_user_active

存储解析清洗之后的用户主动活跃数据，对数据进行去重，并且过滤掉xaid为空的数据。

### （1）源表

```
ods_user_active
```

### （2）建表语句

```
create external table if not exists dwd_mall.dwd_user_active(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    ad_status    tinyint,
    loading_time    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/user_active/';
```

### （3）映射关系：插入数据

```
insert overwrite table dwd_mall.dwd_user_active partition(dt='20220309')  select
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.ad_status') as ad_status,
get_json_object(log,'$.loading_time') as loading_time
from 
(
select log from ods_mall.ods_user_active where dt = '20220309' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
```

## 2、dwd_click_good

### （1）源表

```
ods_click_good
```

### （2）建表语句

```
create external table if not exists dwd_mall.dwd_click_good(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    goods_id    bigint,
    location    tinyint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/click_good/';
```

### （3）映射关系

```
insert overwrite table dwd_mall.dwd_click_good partition(dt='20220309')  select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.goods_id') as goods_id,
get_json_object(log,'$.location') as location
from
(
select log from ods_mall.ods_click_good where dt = '20220309' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
```

## 3、dwd_good_item

### （1）源表

```
ods_good_item
```

### （2）建表语句

```
create external table if not exists dwd_mall.dwd_good_item(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    goods_id    bigint,
    stay_time    bigint,
	loading_time    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/good_item/';
```

### （3）映射关系

```
insert overwrite table dwd_mall.dwd_good_item partition(dt='20220309') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.goods_id') as goods_id,
get_json_object(log,'$.stay_time') as stay_time,
get_json_object(log,'$.loading_time') as loading_time
from
(
select log from ods_mall.ods_good_item where dt = '20220309' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
```

## 4、dwd_good_list

### （1）源表

```
ods_good_list
```

### （2）建表语句

```
create external table if not exists dwd_mall.dwd_good_list(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    loading_time    bigint,
    loading_type    tinyint,
	goods_num    tinyint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/good_list/';
```

### （3）映射关系

```
insert overwrite table dwd_mall.dwd_good_list partition(dt='20220309') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.loading_time') as loading_time,
get_json_object(log,'$.loading_type') as loading_type,
get_json_object(log,'$.goods_num') as goods_num
from
(
select log from ods_mall.ods_good_list where dt = '20220309' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
```

## 5、dwd_app_close

### （1）源表

```
ods_app_close
```

### （2）建表语句

```
create external table if not exists dwd_mall.dwd_app_close(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/app_close/';
```

### （3）映射关系

```
insert overwrite table dwd_mall.dwd_app_close partition(dt='20220309') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime
from
(
select log from ods_mall.ods_app_close where dt = '20220309' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
```

# 四、针对dwd层抽取脚本

## 1、表初始化脚本（初始化执行一次）

dwd_mall_init_table.sh

内容如下：

```
#!/bin/bash
# dwd层数据库和表初始化脚本，只需要执行一次即可

hive -e "
create database if not exists dwd_mall;

create external table if not exists dwd_mall.dwd_user_active(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    ad_status    tinyint,
    loading_time    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/user_active/';




create external table if not exists dwd_mall.dwd_click_good(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    goods_id    bigint,
    location    tinyint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/click_good/';



create external table if not exists dwd_mall.dwd_good_item(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    goods_id    bigint,
    stay_time    bigint,
	loading_time    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/good_item/';



create external table if not exists dwd_mall.dwd_good_list(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint,
    loading_time    bigint,
    loading_type    tinyint,
	goods_num    tinyint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/good_list/';


create external table if not exists dwd_mall.dwd_app_close(
    user_id    bigint,
    xaid    string,
    platform    tinyint,
    ver    string,
    vercode    string,
    net    bigint,
    brand    string,
    model    string,
    display    string,
    osver    string,
    acttime    bigint
)partitioned by(dt string) 
 row format delimited  
 fields terminated by '\t'
 location 'hdfs://bigdata01:9000/data/dwd/app_close/';
"
```

## 2、添加分区数据脚本（每天执行一次）

dwd_mall_add_partition.sh

内容如下：

```
#!/bin/bash
# 基于ods层的表进行清洗，将清洗之后的数据添加到dwd层对应表的对应分区中
# 每天凌晨执行一次

# 默认获取昨天的日期，也支持传参指定一个日期
if [ "z$1" = "z" ]
then
dt=`date +%Y%m%d --date="1 days ago"`
else
dt=$1
fi

hive -e "
insert overwrite table dwd_mall.dwd_user_active partition(dt='${dt}')  select
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.ad_status') as ad_status,
get_json_object(log,'$.loading_time') as loading_time
from 
(
select log from ods_mall.ods_user_active where dt = '${dt}' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';



insert overwrite table dwd_mall.dwd_click_good partition(dt='${dt}')  select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.goods_id') as goods_id,
get_json_object(log,'$.location') as location
from
(
select log from ods_mall.ods_click_good where dt = '${dt}' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';


insert overwrite table dwd_mall.dwd_good_item partition(dt='${dt}') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.goods_id') as goods_id,
get_json_object(log,'$.stay_time') as stay_time,
get_json_object(log,'$.loading_time') as loading_time
from
(
select log from ods_mall.ods_good_item where dt = '${dt}' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';



insert overwrite table dwd_mall.dwd_good_list partition(dt='${dt}') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime,
get_json_object(log,'$.loading_time') as loading_time,
get_json_object(log,'$.loading_type') as loading_type,
get_json_object(log,'$.goods_num') as goods_num
from
(
select log from ods_mall.ods_good_list where dt = '${dt}' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';


insert overwrite table dwd_mall.dwd_app_close partition(dt='${dt}') select 
get_json_object(log,'$.uid') as user_id,
get_json_object(log,'$.xaid') as xaid,
get_json_object(log,'$.platform') as platform,
get_json_object(log,'$.ver') as ver,
get_json_object(log,'$.vercode') as vercode,
get_json_object(log,'$.net') as net,
get_json_object(log,'$.brand') as brand,
get_json_object(log,'$.model') as model,
get_json_object(log,'$.display') as display,
get_json_object(log,'$.osver') as osver,
get_json_object(log,'$.acttime') as acttime
from
(
select log from ods_mall.ods_app_close where dt = '${dt}' group by log
) as tmp
where get_json_object(log,'$.xaid') !='';
"
```

# 五、执行脚本

脚本在目录下
![在这里插入图片描述](https://img-blog.csdnimg.cn/ef3d0bacdc724794a3601ef958542957.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)

执行脚本

```
sh dwd_mall_init_table.sh
sh dwd_mall_add_partition.sh 20220309
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/9e248776c9174e6d93d2b2534edd6add.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/4011c1e7e78c4313b37aab233e8d48ca.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/4cfcd4e76d434373a1771c4075cae97a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)

# 六、验证

连接到hive

```
hive
show databases;
use dwd_mall;
show tables;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2283a14065e3432a975bdb423c9eb9ad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)

检查是否有表和数据。

```
select * from dwd_app_close limit 1;

select * from dwd_click_good limit 1;

select * from dwd_good_item limit 1;

select * from dwd_good_list limit 1;

select * from dwd_user_active limit 1;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/d68c7b2d51cc45f0b84ff24b319b42a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAVFdNXzA1MTg=,size_20,color_FFFFFF,t_70,g_se,x_16)