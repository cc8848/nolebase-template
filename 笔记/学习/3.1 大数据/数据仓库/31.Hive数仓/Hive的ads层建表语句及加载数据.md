- [Hive的ads层建表语句及加载数据_大数据翻身的博客-CSDN博客_ads表](https://blog.csdn.net/qq_56870570/article/details/120331671?spm=1001.2014.3001.5502)

## **建表说明**

ADS层不涉及建模，建表根据具体需求而定。

## **用户主题**

**用户统计**

该需求为用户综合统计，其中包含若干指标，以下为对每个指标的解释说明。

| 指标           | 说明                   | 对应字段             |
| -------------- | ---------------------- | -------------------- |
| 新增用户数     | 统计新增注册用户人数   | new_user_count       |
| 新增下单用户数 | 统计新增下单用户人数   | new_order_user_count |
| 下单总金额     | 统计所有订单总额       | order_final_amount   |
| 下单用户数     | 统计下单用户总数       | order_user_count     |
| 未下单用户数   | 统计活跃但未下单用户数 | no_order_user_count  |

### 建表语句：

```sql
DROP TABLE IF EXISTS ads_user_total;
CREATE EXTERNAL TABLE `ads_user_total` (
  `dt` STRING COMMENT '统计日期',
  `recent_days` BIGINT COMMENT '最近天数,0:累积值,1:最近1天,7:最近7天,30:最近30天',
  `new_user_count` BIGINT COMMENT '新注册用户数',
  `new_order_user_count` BIGINT COMMENT '新增下单用户数',
  `order_final_amount` DECIMAL(16,2) COMMENT '下单总金额',
  `order_user_count` BIGINT COMMENT '下单用户数',
  `no_order_user_count` BIGINT COMMENT '未下单用户数(具体指活跃用户中未下单用户)'
) COMMENT '用户统计'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/ads/ads_user_total/';
```

### 数据装载：

```sql
insert overwrite table ads_user_total
select * from ads_user_total
union
select
    '2020-06-14',
    recent_days,
    sum(if(login_date_first>=recent_days_ago,1,0)) new_user_count,
    sum(if(order_date_first>=recent_days_ago,1,0)) new_order_user_count,
    sum(order_final_amount) order_final_amount,
    sum(if(order_final_amount>0,1,0)) order_user_count,
    sum(if(login_date_last>=recent_days_ago and order_final_amount=0,1,0)) no_order_user_count
from
(
    select
        recent_days,
        user_id,
        login_date_first,
        login_date_last,
        order_date_first,
        case when recent_days=0 then order_final_amount
             when recent_days=1 then order_last_1d_final_amount
             when recent_days=7 then order_last_7d_final_amount
             when recent_days=30 then order_last_30d_final_amount
        end order_final_amount,
        if(recent_days=0,'1970-01-01',date_add('2020-06-14',-recent_days+1)) recent_days_ago
    from dwt_user_topic lateral view explode(Array(0,1,7,30)) tmp as recent_days
    where dt='2020-06-14'
)t1
group by recent_days;
```

## **各地区订单统计**

该需求包含各省份订单总数和订单总金额。

### 建表语句：

```sql
DROP TABLE IF EXISTS ads_order_by_province;
CREATE EXTERNAL TABLE `ads_order_by_province` (
  `dt` STRING COMMENT '统计日期',
  `recent_days` BIGINT COMMENT '最近天数,1:最近1天,7:最近7天,30:最近30天',
  `province_id` STRING COMMENT '省份ID',
  `province_name` STRING COMMENT '省份名称',
  `area_code` STRING COMMENT '地区编码',
  `iso_code` STRING COMMENT '国际标准地区编码',
  `iso_code_3166_2` STRING COMMENT '国际标准地区编码',
  `order_count` BIGINT COMMENT '订单数',
  `order_amount` DECIMAL(16,2) COMMENT '订单金额'
) COMMENT '各地区订单统计'
ROW FORMAT DELIMITED  FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/ads/ads_order_by_province/';
```

### 数据装载：

```sql
insert overwrite table ads_order_by_province
select * from ads_order_by_province
union
select
    dt,
    recent_days,
    province_id,
    province_name,
    area_code,
    iso_code,
    iso_3166_2,
    order_count,
    order_amount
from
(
    select
        '2020-06-14' dt,
        recent_days,
        province_id,
        sum(order_count) order_count,
        sum(order_amount) order_amount
    from
    (
        select
            recent_days,
            province_id,
            case
                when recent_days=1 then order_last_1d_count
                when recent_days=7 then order_last_7d_count
                when recent_days=30 then order_last_30d_count
            end order_count,
            case
                when recent_days=1 then order_last_1d_final_amount
                when recent_days=7 then order_last_7d_final_amount
                when recent_days=30 then order_last_30d_final_amount
            end order_amount
        from dwt_area_topic lateral view explode(Array(1,7,30)) tmp as recent_days
        where dt='2020-06-14'
    )t1
    group by recent_days,province_id
)t2
join dim_base_province t3
on t2.province_id=t3.id;
```

## 用户留存率

留存分析一般包含新增留存和活跃留存分析。

新增留存分析是分析某天的新增用户中，有多少人有后续的活跃行为。活跃留存分析是分析某天的活跃用户中，有多少人有后续的活跃行为。

留存分析是衡量产品对用户价值高低的重要指标。

此处要求统计新增留存率，新增留存率具体是指留存用户数与新增用户数的比值，例如2020-06-14新增100个用户，1日之后（2020-06-15）这100人中有80个人活跃了，那2020-06-14的1日留存数则为80，2020-06-14的1日留存率则为80%。

要求统计每天的1至7日留存率，如下图所示。

![img](https://img-blog.csdnimg.cn/20210916162411622.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5aSn5pWw5o2u57-76Lqr,size_20,color_FFFFFF,t_70,g_se,x_16)

### 建表语句：

```sql
DROP TABLE IF EXISTS ads_user_retention;
CREATE EXTERNAL TABLE ads_user_retention (
  `dt` STRING COMMENT '统计日期',
  `create_date` STRING COMMENT '用户新增日期',
  `retention_day` BIGINT COMMENT '截至当前日期留存天数',
  `retention_count` BIGINT COMMENT '留存用户数量',
  `new_user_count` BIGINT COMMENT '新增用户数量',
  `retention_rate` DECIMAL(16,2) COMMENT '留存率'
) COMMENT '用户留存率'
ROW FORMAT DELIMITED  FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/ads/ads_user_retention/';
```

###  数据装载：

```sql
insert overwrite table ads_user_retention
select * from ads_user_retention
union
select
    '2020-06-14',
    login_date_first create_date,
    datediff('2020-06-14',login_date_first) retention_day,
    sum(if(login_date_last='2020-06-14',1,0)) retention_count,
    count(*) new_user_count,
    cast(sum(if(login_date_last='2020-06-14',1,0))/count(*)*100 as decimal(16,2)) retention_rate
from dwt_user_topic
where dt='2020-06-14'
and login_date_first>=date_add('2020-06-14',-7)
and login_date_first<'2020-06-14'
group by login_date_first;
```

## 订单主题

### 订单统计

该需求包含订单总数，订单总金额和下单总人数。

### 建表语句

```sql
DROP TABLE IF EXISTS ads_order_total;
CREATE EXTERNAL TABLE `ads_order_total` (
  `dt` STRING COMMENT '统计日期',
  `recent_days` BIGINT COMMENT '最近天数,1:最近1天,7:最近7天,30:最近30天',
  `order_count` BIGINT COMMENT '订单数',
  `order_amount` DECIMAL(16,2) COMMENT '订单金额',
  `order_user_count` BIGINT COMMENT '下单人数'
) COMMENT '订单统计'
ROW FORMAT DELIMITED  FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/ads/ads_order_total/';
```

### 数据装载

```sql
insert overwrite table ads_order_total
select * from ads_order_total
union
select
    '2020-06-14',
    recent_days,
    sum(order_count),
    sum(order_final_amount) order_final_amount,
    sum(if(order_final_amount>0,1,0)) order_user_count
from
(
    select
        recent_days,
        user_id,
        case when recent_days=0 then order_count
             when recent_days=1 then order_last_1d_count
             when recent_days=7 then order_last_7d_count
             when recent_days=30 then order_last_30d_count
        end order_count,
        case when recent_days=0 then order_final_amount
             when recent_days=1 then order_last_1d_final_amount
             when recent_days=7 then order_last_7d_final_amount
             when recent_days=30 then order_last_30d_final_amount
        end order_final_amount
    from dwt_user_topic lateral view explode(Array(1,7,30)) tmp as recent_days
    where dt='2020-06-14'
)t1
group by recent_days;
```

## 品牌复购率

品牌复购率是指一段时间内重复购买某品牌的人数与购买过该品牌的人数的比值。重复购买即购买次数大于等于2，购买过即购买次数大于1。

此处要求统计最近1,7,30天的各品牌复购率。

### 建表语句：

```bash
DROP TABLE IF EXISTS ads_repeat_purchase;
CREATE EXTERNAL TABLE `ads_repeat_purchase` (
  `dt` STRING COMMENT '统计日期',
  `recent_days` BIGINT COMMENT '最近天数,1:最近1天,7:最近7天,30:最近30天',
  `tm_id` STRING COMMENT '品牌ID',
  `tm_name` STRING COMMENT '品牌名称',
  `order_repeat_rate` DECIMAL(16,2) COMMENT '复购率'
) COMMENT '品牌复购率'
ROW FORMAT DELIMITED  FIELDS TERMINATED BY '\t'
LOCATION '/warehouse/gmall/ads/ads_repeat_purchase/';
```

### ***\*数据装载\****

***\*思路分析：该需求可分两步实现：\****

***\*第一步：统计每个用户购买每个品牌的次数。\****

***\*第二步：分别统计购买次数大于1的人数和大于2的人数。\****

```sql
insert overwrite table ads_repeat_purchase
select * from ads_repeat_purchase
union
select
    '2020-06-14' dt,
    recent_days,
    tm_id,
    tm_name,
    cast(sum(if(order_count>=2,1,0))/sum(if(order_count>=1,1,0))*100 as decimal(16,2))
from
(
    select
        recent_days,
        user_id,
        tm_id,
        tm_name,
        sum(order_count) order_count
    from
    (
        select
            recent_days,
            user_id,
            sku_id,
            count(*) order_count
        from dwd_order_detail lateral view explode(Array(1,7,30)) tmp as recent_days
        where dt>=date_add('2020-06-14',-29)
        and dt>=date_add('2020-06-14',-recent_days+1)
        group by recent_days, user_id,sku_id
    )t1
    left join
    (
        select
            id,
            tm_id,
            tm_name
        from dim_sku_info
        where dt='2020-06-14'
    )t2
    on t1.sku_id=t2.id
    group by recent_days,user_id,tm_id,tm_name
)t3
group by recent_days,tm_id,tm_name;
```

##  **ADS层业务数据导入脚本**

***\*编写及执行脚本\****

（1）在/home/muzili/bin目录下创建脚本dwt_to_ads.sh

[muzili@hadoop102 bin]$ vim dwt_to_ads.sh

在脚本中填写如下内容:

```bash
#!/bin/bash
 
APP=gmall
 
# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$2" ] ;then
    do_date=$2
else 
    do_date=`date -d "-1 day" +%F`
fi
 
ads_order_by_province="
insert overwrite table ${APP}.ads_order_by_province
select * from ${APP}.ads_order_by_province
union
select
    dt,
    recent_days,
    province_id,
    province_name,
    area_code,
    iso_code,
    iso_3166_2,
    order_count,
    order_amount
from
(
    select
        '$do_date' dt,
        recent_days,
        province_id,
        sum(order_count) order_count,
        sum(order_amount) order_amount
    from
    (
        select
            recent_days,
            province_id,
            case
                when recent_days=1 then order_last_1d_count
                when recent_days=7 then order_last_7d_count
                when recent_days=30 then order_last_30d_count
            end order_count,
            case
                when recent_days=1 then order_last_1d_final_amount
                when recent_days=7 then order_last_7d_final_amount
                when recent_days=30 then order_last_30d_final_amount
            end order_amount
        from ${APP}.dwt_area_topic lateral view explode(Array(1,7,30)) tmp as recent_days
        where dt='$do_date'
    )t1
    group by recent_days,province_id
)t2
join ${APP}.dim_base_province t3
on t2.province_id=t3.id;
"
 
ads_order_total="
insert overwrite table ${APP}.ads_order_total
select * from ${APP}.ads_order_total
union
select
    '$do_date',
    recent_days,
    sum(order_count),
    sum(order_final_amount) order_final_amount,
    sum(if(order_final_amount>0,1,0)) order_user_count
from
(
    select
        recent_days,
        user_id,
        case when recent_days=0 then order_count
             when recent_days=1 then order_last_1d_count
             when recent_days=7 then order_last_7d_count
             when recent_days=30 then order_last_30d_count
        end order_count,
        case when recent_days=0 then order_final_amount
             when recent_days=1 then order_last_1d_final_amount
             when recent_days=7 then order_last_7d_final_amount
             when recent_days=30 then order_last_30d_final_amount
        end order_final_amount
    from ${APP}.dwt_user_topic lateral view explode(Array(1,7,30)) tmp as recent_days
    where dt='$do_date'
)t1
group by recent_days;
"
 
ads_repeat_purchase="
insert overwrite table ${APP}.ads_repeat_purchase
select * from ${APP}.ads_repeat_purchase
union
select
    '$do_date' dt,
    recent_days,
    tm_id,
    tm_name,
    cast(sum(if(order_count>=2,1,0))/sum(if(order_count>=1,1,0))*100 as decimal(16,2))
from
(
    select
        recent_days,
        user_id,
        tm_id,
        tm_name,
        sum(order_count) order_count
    from
    (
        select
            recent_days,
            user_id,
            sku_id,
            count(*) order_count
        from ${APP}.dwd_order_detail lateral view explode(Array(1,7,30)) tmp as recent_days
        where dt>=date_add('$do_date',-29)
        and dt>=date_add('$do_date',-recent_days+1)
        group by recent_days, user_id,sku_id
    )t1
    left join
    (
        select
            id,
            tm_id,
            tm_name
        from ${APP}.dim_sku_info
        where dt='$do_date'
    )t2
    on t1.sku_id=t2.id
    group by recent_days,user_id,tm_id,tm_name
)t3
group by recent_days,tm_id,tm_name;
"
 
ads_user_retention="
insert overwrite table ${APP}.ads_user_retention
select * from ${APP}.ads_user_retention
union
select
    '$do_date',
    login_date_first create_date,
    datediff('$do_date',login_date_first) retention_day,
    sum(if(login_date_last='$do_date',1,0)) retention_count,
    count(*) new_user_count,
    cast(sum(if(login_date_last='$do_date',1,0))/count(*)*100 as decimal(16,2)) retention_rate
from ${APP}.dwt_user_topic
where dt='$do_date'
and login_date_first>=date_add('$do_date',-7)
and login_date_first<'$do_date'
group by login_date_first;
"
 
ads_user_total="
insert overwrite table ${APP}.ads_user_total
select * from ${APP}.ads_user_total
union
select
    '$do_date',
    recent_days,
    sum(if(login_date_first>=recent_days_ago,1,0)) new_user_count,
    sum(if(order_date_first>=recent_days_ago,1,0)) new_order_user_count,
    sum(order_final_amount) order_final_amount,
    sum(if(order_final_amount>0,1,0)) order_user_count,
    sum(if(login_date_last>=recent_days_ago and order_final_amount=0,1,0)) no_order_user_count
from
(
    select
        recent_days,
        user_id,
        login_date_first,
        login_date_last,
        order_date_first,
        case when recent_days=0 then order_final_amount
             when recent_days=1 then order_last_1d_final_amount
             when recent_days=7 then order_last_7d_final_amount
             when recent_days=30 then order_last_30d_final_amount
        end order_final_amount,
        if(recent_days=0,'1970-01-01',date_add('$do_date',-recent_days+1)) recent_days_ago
    from ${APP}.dwt_user_topic lateral view explode(Array(0,1,7,30)) tmp as recent_days
    where dt='$do_date'
)t1
group by recent_days;
"
 
case $1 in
    
    "ads_order_by_province" )
        hive -e "$ads_order_by_province" 
    ;;
    "ads_order_total" )
        hive -e "$ads_order_total" 
    ;;
    "ads_repeat_purchase" )
        hive -e "$ads_repeat_purchase" 
    ;;
    "ads_user_retention" )
        hive -e "$ads_user_retention" 
    ;;
    "ads_user_total" )
        hive -e "$ads_user_total" 
    ;;
    "all" )
        hive -e "$ads_order_by_province$ads_order_total$ads_repeat_purchase$ads_user_retention$ads_user_total"
    ;;
esac
```

（2）增加脚本执行权限

[muzili@hadoop102 bin]$ chmod 777 dwt_to_ads.sh

（3）执行脚本

[muzili@hadoop102 bin]$ dwt_to_ads.sh all 2021-09-16                                   

（4）查看数据是否导入