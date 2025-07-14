- [数据仓库搭建ADS层_一个人的牛牛的博客-CSDN博客_ads层](https://blog.csdn.net/qq_55906442/article/details/125008169)

# 搭建[ADS](https://so.csdn.net/so/search?q=ADS&spm=1001.2101.3001.7020)层

ADS层不涉及建模，建表根据具体需求而定。

## 一、设备主题

### 1.1活跃设备数（日、周、月）

需求定义：

日活：当日活跃的设备数

周活：当周活跃的设备数

月活：当月活跃的设备数

1）建表语句

```typescript
create external table ads_uv_count(
    `dt` string COMMENT '统计日期',
    `day_count` bigint COMMENT '当日用户数量',
    `wk_count`  bigint COMMENT '当周用户数量',
    `mn_count`  bigint COMMENT '当月用户数量',
    `is_weekend` string COMMENT 'Y,N是否是周末,用于得到本周最终结果',
    `is_monthend` string COMMENT 'Y,N是否是月末,用于得到本月最终结果' 
) COMMENT '活跃设备数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_uv_count/';
```

2）导入数据

```sql
insert into table ads_uv_count 
select  
    '2022-05-20' dt,
    daycount.ct,
    wkcount.ct,
    mncount.ct,
    if(date_add(next_day('2022-05-20','MO'),-1)='2022-05-20','Y','N') ,
    if(last_day('2022-05-20')='2022-05-20','Y','N') 
from 
(
    select  
        '2022-05-20' dt,
        count(*) ct
    from dwt_uv_topic
    where login_date_last='2022-05-20'  
)daycount join 
( 
    select  
        '2022-05-20' dt,
        count (*) ct
    from dwt_uv_topic
    where login_date_last>=date_add(next_day('2022-05-20','MO'),-7) 
    and login_date_last<= date_add(next_day('2022-05-20','MO'),-1) 
) wkcount on daycount.dt=wkcount.dt
join 
( 
    select  
        '2022-05-20' dt,
        count (*) ct
    from dwt_uv_topic
    where date_format(login_date_last,'yyyy-MM')=date_format('2022-05-20','yyyy-MM')  
)mncount on daycount.dt=mncount.dt;
```

3）查询导入结果

```sql
select * from ads_uv_count;
```

![img](https://img-blog.csdnimg.cn/390a6f15160a4ac68aff12ab04bf64fc.png)



### 1.2 每日新增设备

1）建表语句

```sql
create external table ads_new_mid_count
(
`create_date` string comment '创建时间' ,
`new_mid_count` BIGINT comment '新增设备数量' 
)  COMMENT '每日新增设备数量'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_new_mid_count/';
```

2）导入数据

```sql
insert into table ads_new_mid_count 
select
    '2022-05-20',
    count(*)
from dwt_uv_topic
where login_date_first='2022-05-20';
```

3）查询导入结果

```sql
select * from ads_new_mid_count;
```

![img](https://img-blog.csdnimg.cn/47ab2d0e7170494f9ca6dfaa1bffecb5.png)



### 1.3留存率

![img](https://img-blog.csdnimg.cn/87ccc299771d41da8c708ccf2ee0445b.png)

1）建表语句

```typescript
create external table ads_user_retention_day_rate 
(
`stat_date` string comment '统计日期',
`create_date` string  comment '设备新增日期',
`retention_day` int comment '截止当前日期留存天数',
`retention_count` bigint comment  '留存数量',
`new_mid_count` bigint comment '设备新增数量',
`retention_ratio` decimal(16,2) comment '留存率'
)  COMMENT '留存率'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_user_retention_day_rate/';
```

2）导入数据

```sql
insert into table ads_user_retention_day_rate
select
    '2022-05-20',
    date_add('2022-05-20',-1),
    1,--留存天数
    sum(if(login_date_first=date_add('2022-05-20',-1) and login_date_last='2022-05-20',1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-1),1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-1) and login_date_last='2022-05-20',1,0))/sum(if(login_date_first=date_add('2022-05-20',-1),1,0))*100
from dwt_uv_topic
 
union all
 
select
    '2022-05-20',
    date_add('2022-05-20',-2),
    2,
    sum(if(login_date_first=date_add('2022-05-20',-2) and login_date_last='2022-05-20',1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-2),1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-2) and login_date_last='2022-05-20',1,0))/sum(if(login_date_first=date_add('2022-05-20',-2),1,0))*100
from dwt_uv_topic
 
union all
 
select
    '2022-05-20',
    date_add('2022-05-20',-3),
    3,
    sum(if(login_date_first=date_add('2022-05-20',-3) and login_date_last='2022-05-20',1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-3),1,0)),
    sum(if(login_date_first=date_add('2022-05-20',-3) and login_date_last='2022-05-20',1,0))/sum(if(login_date_first=date_add('2022-05-20',-3),1,0))*100
from dwt_uv_topic;
```

3）查询导入结果

```sql
select * from ads_user_retention_day_rate;
```

![img](https://img-blog.csdnimg.cn/1c8cf07a6a6d42fb84cd6336b4e182e0.png)

### 1.4沉默用户数

1）建表语句

```sql
create external table ads_silent_count( 
    `dt` string COMMENT '统计日期',
    `silent_count` bigint COMMENT '沉默设备数'
) COMMENT '沉默用户数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_silent_count';
```

2）导入数据（2022-05-20）

```sql
insert into table ads_silent_count
select
    '2022-05-20',
    count(*) 
from dwt_uv_topic
where login_date_first=login_date_last
and login_date_last<=date_add('2022-05-20',-7);
```

3）查询导入结果

```sql
select * from ads_silent_count;
```

![img](https://img-blog.csdnimg.cn/84405b5d4cc84b1a9ee730d3ff16c8be.png)



### 1.5本周回流用户数

需求定义：

本周回流用户：上周未活跃，本周活跃的设备，且不是本周新增设备

1）建表语句

```typescript
create external table ads_back_count( 
    `dt` string COMMENT '统计日期',
    `wk_dt` string COMMENT '统计日期所在周',
    `wastage_count` bigint COMMENT '回流设备数'
) COMMENT '本周回流用户数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_back_count';
```

2）导入数据

```sql
insert into table ads_back_count
select
    '2022-05-20',
    concat(date_add(next_day('2022-05-20','MO'),-7),'_', date_add(next_day('2022-05-20','MO'),-1)),
    count(*)
from
(
    select
        mid_id
    from dwt_uv_topic
    where login_date_last>=date_add(next_day('2022-05-20','MO'),-7) 
    and login_date_last<= date_add(next_day('2022-05-20','MO'),-1)
    and login_date_first<date_add(next_day('2022-05-20','MO'),-7)
)current_wk
left join
(
    select
        mid_id
    from dws_uv_detail_daycount
    where dt>=date_add(next_day('2022-05-20','MO'),-7*2) 
    and dt<= date_add(next_day('2022-05-20','MO'),-7-1) 
    group by mid_id
)last_wk
on current_wk.mid_id=last_wk.mid_id
where last_wk.mid_id is null;
```

3）查询导入结果

```sql
select * from ads_back_count;
```

![img](https://img-blog.csdnimg.cn/6720049f894d4c569f3e9ff865dea6b4.png)



### 1.6流失用户数

需求定义：

流失用户：最近7天未活跃的设备

1）建表语句

```sql
create external table ads_wastage_count( 
    `dt` string COMMENT '统计日期',
    `wastage_count` bigint COMMENT '流失设备数'
) COMMENT '流失用户数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_wastage_count';
```

2）导入数据(2022-05-20)

```sql
insert into table ads_wastage_count
select
     '2022-05-20',
     count(*)
from 
(
    select 
        mid_id
    from dwt_uv_topic
    where login_date_last<=date_add('2022-05-20',-7)
    group by mid_id
)t1;
```

3）查询导入结果

```sql
select * from ads_wastage_count;
```

![img](https://img-blog.csdnimg.cn/d698db157ba14418bfde78989e474837.png)



### 1.7最近连续三周活跃用户数

1）建表语句

```typescript
create external table ads_continuity_wk_count( 
    `dt` string COMMENT '统计日期,一般用结束周周日日期,如果每天计算一次,可用当天日期',
    `wk_dt` string COMMENT '持续时间',
    `continuity_count` bigint COMMENT '活跃用户数'
) COMMENT '最近连续三周活跃用户数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_continuity_wk_count';
```

2）导入数据（2022-05-20所在周）

```sql
insert into table ads_continuity_wk_count
select
    '2022-05-20',
    concat(date_add(next_day('2022-05-20','MO'),-7*3),'_',date_add(next_day('2022-05-20','MO'),-1)),
    count(*)
from
(
    select
        mid_id
    from
    (
        select
            mid_id
        from dws_uv_detail_daycount
        where dt>=date_add(next_day('2022-05-20','monday'),-7)
        and dt<=date_add(next_day('2022-05-20','monday'),-1)
        group by mid_id
 
        union all
 
        select
            mid_id
        from dws_uv_detail_daycount
        where dt>=date_add(next_day('2022-05-20','monday'),-7*2)
        and dt<=date_add(next_day('2022-05-20','monday'),-7-1)
        group by mid_id
 
        union all
 
        select
            mid_id
        from dws_uv_detail_daycount
        where dt>=date_add(next_day('2022-05-20','monday'),-7*3)
        and dt<=date_add(next_day('2022-05-20','monday'),-7*2-1)
        group by mid_id
    )t1
    group by mid_id
    having count(*)=3
)t2;
```

3）查询导入结果

```sql
select * from ads_continuity_wk_count;
```

![img](https://img-blog.csdnimg.cn/334789a46574413fb4457a9e4253b471.png)



### 1.8最近七天内连续三天活跃用户数

1）建表语句

```typescript
create external table ads_continuity_uv_count( 
    `dt` string COMMENT '统计日期',
    `wk_dt` string COMMENT '最近7天日期',
    `continuity_count` bigint
) COMMENT '最近七天内连续三天活跃用户数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_continuity_uv_count';
```

2）导入数据

```sql
insert into table ads_continuity_uv_count
select
    '2022-05-20',
    concat(date_add('2022-05-20',-6),'_','2022-05-20'),
    count(*)
from
(
    select mid_id
    from
    (
        select mid_id
        from
        (
            select 
                mid_id,
                date_sub(dt,rank) date_dif
            from
            (
                select
                    mid_id,
                    dt,
                    rank() over(partition by mid_id order by dt) rank
                from dws_uv_detail_daycount
                where dt>=date_add('2022-05-20',-6) and dt<='2022-05-20'
            )t1
        )t2 
        group by mid_id,date_dif
        having count(*)>=3
    )t3 
    group by mid_id
)t4;
```

3）查询导入结果

```sql
select * from ads_continuity_uv_count;
```

![img](https://img-blog.csdnimg.cn/635a2b163c604c03a344e7fcbfbfbf45.png)





## 二、会员主题

### 2.1会员信息

1）建表语句

```typescript
create external table ads_user_topic(
    `dt` string COMMENT '统计日期',
    `day_users` string COMMENT '活跃会员数',
    `day_new_users` string COMMENT '新增会员数',
    `day_new_payment_users` string COMMENT '新增消费会员数',
    `payment_users` string COMMENT '总付费会员数',
    `users` string COMMENT '总会员数',
    `day_users2users` decimal(16,2) COMMENT '会员活跃率',
    `payment_users2users` decimal(16,2) COMMENT '会员付费率',
    `day_new_users2users` decimal(16,2) COMMENT '会员新鲜度'
) COMMENT '会员信息表'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_user_topic';
```

2）导入数据

```sql
insert into table ads_user_topic
select
    '2022-05-20',
    sum(if(login_date_last='2022-05-20',1,0)),
    sum(if(login_date_first='2022-05-20',1,0)),
    sum(if(payment_date_first='2022-05-20',1,0)),
    sum(if(payment_count>0,1,0)),
    count(*),
    sum(if(login_date_last='2022-05-20',1,0))/count(*),
    sum(if(payment_count>0,1,0))/count(*),
    sum(if(login_date_first='2022-05-20',1,0))/sum(if(login_date_last='2022-05-20',1,0))
from dwt_user_topic;
```

3）查询导入结果

```sql
select * from ads_user_topic;
```

![img](https://img-blog.csdnimg.cn/162c2e93911f48d7ae6180a327d381a3.png)



### 2.2漏斗分析

统计“浏览首页->浏览商品详情页->加入购物车->下单->支付”的转化率

思路：统计各个行为的人数，然后计算比值。

1）建表语句

```typescript
create external  table ads_user_action_convert_day(
    `dt` string COMMENT '统计日期',
    `home_count`  bigint COMMENT '浏览首页人数',
    `good_detail_count` bigint COMMENT '浏览商品详情页人数',
    `home2good_detail_convert_ratio` decimal(16,2) COMMENT '首页到商品详情转化率',
    `cart_count` bigint COMMENT '加入购物车的人数',
    `good_detail2cart_convert_ratio` decimal(16,2) COMMENT '商品详情页到加入购物车转化率',
    `order_count` bigint     COMMENT '下单人数',
    `cart2order_convert_ratio`  decimal(16,2) COMMENT '加入购物车到下单转化率',
    `payment_amount` bigint     COMMENT '支付人数',
    `order2payment_convert_ratio` decimal(16,2) COMMENT '下单到支付的转化率'
) COMMENT '漏斗分析'
row format delimited  fields terminated by '\t'
location '/warehouse/gmall/ads/ads_user_action_convert_day/';
```

2）导入数据

```sql
with
tmp_uv as
(
    select
        '2022-05-20' dt,
        sum(if(array_contains(pages,'home'),1,0)) home_count,
        sum(if(array_contains(pages,'good_detail'),1,0)) good_detail_count
    from
    (
        select
            mid_id,
            collect_set(page_id) pages
        from dwd_page_log
        where dt='2022-05-20'
        and page_id in ('home','good_detail')
        group by mid_id
    )tmp
),
tmp_cop as
(
    select 
        '2022-05-20' dt,
        sum(if(cart_count>0,1,0)) cart_count,
        sum(if(order_count>0,1,0)) order_count,
        sum(if(payment_count>0,1,0)) payment_count
    from dws_user_action_daycount
    where dt='2022-05-20'
)
insert into table ads_user_action_convert_day
select
    tmp_uv.dt,
    tmp_uv.home_count,
    tmp_uv.good_detail_count,
    tmp_uv.good_detail_count/tmp_uv.home_count*100,
    tmp_cop.cart_count,
    tmp_cop.cart_count/tmp_uv.good_detail_count*100,
    tmp_cop.order_count,
    tmp_cop.order_count/tmp_cop.cart_count*100,
    tmp_cop.payment_count,
    tmp_cop.payment_count/tmp_cop.order_count*100
from tmp_uv
join tmp_cop
on tmp_uv.dt=tmp_cop.dt;
```

3）查询导入结果

```sql
select * from ads_user_action_convert_day;
```

![img](https://img-blog.csdnimg.cn/3eea6b7814ec4ba596ebff6447e395ad.png)



## 三、商品主题

### 3.1商品主题

1）建表语句

```typescript
create external table ads_product_info(
    `dt` string COMMENT '统计日期',
    `sku_num` string COMMENT 'sku个数',
    `spu_num` string COMMENT 'spu个数'
) COMMENT '商品个数信息'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_product_info';
```

2）导入数据

```sql
insert into table ads_product_info
select
    '2022-05-20' dt,
    sku_num,
    spu_num
from
(
    select
        '2022-05-20' dt,
        count(*) sku_num
    from
        dwt_sku_topic
) tmp_sku_num
join
(
    select
        '2022-05-20' dt,
        count(*) spu_num
    from
    (
        select
            spu_id
        from
            dwt_sku_topic
        group by
            spu_id
    ) tmp_spu_id
) tmp_spu_num
on tmp_sku_num.dt=tmp_spu_num.dt;
```

3）查询导入结果

```sql
select * from ads_product_info;
```

 ![img](https://img-blog.csdnimg.cn/b3577cc9ffa1425695a507b9f46195fa.png)



### 3.2商品营销排名

1）建表语句

```typescript
create external table ads_product_sale_topN(
    `dt` string COMMENT '统计日期',
    `sku_id` string COMMENT '商品ID',
    `payment_amount` bigint COMMENT '销量'
) COMMENT '商品销量排名'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_product_sale_topN';
```

2）导入数据

```sql
insert into table ads_product_sale_topN
select
    '2022-05-20' dt,
    sku_id,
    payment_amount
from
    dws_sku_action_daycount
where
    dt='2022-05-20'
order by payment_amount desc
limit 10;
```

3）查询导入结果

```sql
select * from ads_product_sale_topN;
```

![img](https://img-blog.csdnimg.cn/b4eddbf872f04309a5f99ab097a5cc9d.png)



### 3.3商品收藏排名

1）建表语句

```typescript
create external table ads_product_favor_topN(
    `dt` string COMMENT '统计日期',
    `sku_id` string COMMENT '商品ID',
    `favor_count` bigint COMMENT '收藏量'
) COMMENT '商品收藏排名'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_product_favor_topN';
```

 2）导入数据

```sql
insert into table ads_product_favor_topN
select
    '2022-05-20' dt,
    sku_id,
    favor_count
from
    dws_sku_action_daycount
where
    dt='2022-05-20'
order by favor_count desc
limit 10;
```

3）查询导入结果

```sql
select * from ads_product_favor_topN;
```

![img](https://img-blog.csdnimg.cn/c075e1b5fe2f4a70bc07a27b98131176.png)



### 3.4商品加入购物车排名

1）建表语句

```typescript
create external table ads_product_cart_topN(
    `dt` string COMMENT '统计日期',
    `sku_id` string COMMENT '商品ID',
    `cart_count` bigint COMMENT '加入购物车次数'
) COMMENT '商品加入购物车排名'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_product_cart_topN';
```

2）导入数据

```sql
insert into table ads_product_cart_topN
select
    '2022-05-20' dt,
    sku_id,
    cart_count
from
    dws_sku_action_daycount
where
    dt='2022-05-20'
order by cart_count desc
limit 10;
```

3）查询导入结果

```sql
select * from ads_product_cart_topN;
```

![img](https://img-blog.csdnimg.cn/d898cfb5b83f47f1af4e725770db6d72.png)



### 3.5商品退款率排名（30天）

1）建表语句

```typescript
create external table ads_product_refund_topN(
    `dt` string COMMENT '统计日期',
    `sku_id` string COMMENT '商品ID',
    `refund_ratio` decimal(16,2) COMMENT '退款率'
) COMMENT '商品退款率排名'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_product_refund_topN';
```

2）导入数据

```sql
insert into table ads_product_refund_topN
select
    '2022-05-20',
    sku_id,
    refund_last_30d_count/payment_last_30d_count*100 refund_ratio
from dwt_sku_topic
order by refund_ratio desc
limit 10;
```

3）查询导入结果

```sql
select * from ads_product_refund_topN;
```

![img](https://img-blog.csdnimg.cn/5b9518aec2ba461ea6221bd320fdd346.png)



### 3.6商品差评率

1）建表语句

```typescript
create external table ads_appraise_bad_topN(
    `dt` string COMMENT '统计日期',
    `sku_id` string COMMENT '商品ID',
    `appraise_bad_ratio` decimal(16,2) COMMENT '差评率'
) COMMENT '商品差评率'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_appraise_bad_topN';
```

2）导入数据

```sql
insert into table ads_appraise_bad_topN
select
    '2022-05-20' dt,
    sku_id,
appraise_bad_count/(appraise_good_count+appraise_mid_count+appraise_bad_count+appraise_default_count) appraise_bad_ratio
from
    dws_sku_action_daycount
where
    dt='2022-05-20'
order by appraise_bad_ratio desc
limit 10;
```

3）查询导入结果

```sql
select * from ads_appraise_bad_topN;
```

![img](https://img-blog.csdnimg.cn/586ad5e8a59e4697b1bd11dd66b0f155.png)



## 四、营销主题

### 4.1下单数目统计

需求分析：统计每日下单数，下单金额及下单用户数。

1）建表语句

```sql
create external table ads_order_daycount(
    dt string comment '统计日期',
    order_count bigint comment '单日下单笔数',
    order_amount bigint comment '单日下单金额',
    order_users bigint comment '单日下单用户数'
) comment '下单数目统计'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_order_daycount';
```

2）导入数据

```sql
insert into table ads_order_daycount
select
    '2022-05-20',
    sum(order_count),
    sum(order_amount),
    sum(if(order_count>0,1,0))
from dws_user_action_daycount
where dt='2022-05-20';
```

3）查询导入结果

```sql
select * from ads_order_daycount;
```

![img](https://img-blog.csdnimg.cn/fe4b6deea4e444b88c07b80b09552a8f.png)



### 4.2支付信息统计

1）建表语句

```sql
create external table ads_payment_daycount(
    dt string comment '统计日期',
    order_count bigint comment '单日支付笔数',
    order_amount bigint comment '单日支付金额',
    payment_user_count bigint comment '单日支付人数',
    payment_sku_count bigint comment '单日支付商品数',
    payment_avg_time decimal(16,2) comment '下单到支付的平均时长，取分钟数'
) comment '支付信息统计'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_payment_daycount';
```

2）导入数据

```sql
insert into table ads_payment_daycount
select
    tmp_payment.dt,
    tmp_payment.payment_count,
    tmp_payment.payment_amount,
    tmp_payment.payment_user_count,
    tmp_skucount.payment_sku_count,
    tmp_time.payment_avg_time
from
(
    select
        '2022-05-20' dt,
        sum(payment_count) payment_count,
        sum(payment_amount) payment_amount,
        sum(if(payment_count>0,1,0)) payment_user_count
    from dws_user_action_daycount
    where dt='2022-05-20'
)tmp_payment
join
(
    select
        '2022-05-20' dt,
        sum(if(payment_count>0,1,0)) payment_sku_count 
    from dws_sku_action_daycount
    where dt='2022-05-20'
)tmp_skucount on tmp_payment.dt=tmp_skucount.dt
join
(
    select
        '2022-05-20' dt,
        sum(unix_timestamp(payment_time)-unix_timestamp(create_time))/count(*)/60 payment_avg_time
    from dwd_fact_order_info
    where dt='2022-05-20'
    and payment_time is not null
)tmp_time on tmp_payment.dt=tmp_time.dt;
```

3）查询导入结果

```sql
 select * from ads_payment_daycount;
```

![img](https://img-blog.csdnimg.cn/9dbf44c21edc4893aa7161e2b57e0e8f.png)



### 4.3 品牌复购率统计

1）建表语句

```sql
create external table ads_sale_tm_category1_stat_mn
(  
    tm_id string comment '品牌id',
    category1_id string comment '1级品类id ',
    category1_name string comment '1级品类名称 ',
    buycount   bigint comment  '购买人数',
    buy_twice_last bigint  comment '两次以上购买人数',
    buy_twice_last_ratio decimal(16,2)  comment  '单次复购率',
    buy_3times_last   bigint comment   '三次以上购买人数',
    buy_3times_last_ratio decimal(16,2)  comment  '多次复购率',
    stat_mn string comment '统计月份',
    stat_date string comment '统计日期' 
) COMMENT '品牌复购率统计'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_sale_tm_category1_stat_mn/';
```

2）导入数据

```sql
with 
tmp_order as
(
    select
        user_id,
        order_stats_struct.sku_id sku_id,
        order_stats_struct.order_count order_count
    from dws_user_action_daycount lateral view explode(order_detail_stats) tmp as order_stats_struct
    where date_format(dt,'yyyy-MM')=date_format('2022-05-20','yyyy-MM')
),
tmp_sku as
(
    select
        id,
        tm_id,
        category1_id,
        category1_name
    from dwd_dim_sku_info
    where dt='2022-05-20'
)
insert into table ads_sale_tm_category1_stat_mn
select
    tm_id,
    category1_id,
    category1_name,
    sum(if(order_count>=1,1,0)) buycount,
    sum(if(order_count>=2,1,0)) buyTwiceLast,
    sum(if(order_count>=2,1,0))/sum( if(order_count>=1,1,0)) buyTwiceLastRatio,
    sum(if(order_count>=3,1,0))  buy3timeLast  ,
    sum(if(order_count>=3,1,0))/sum( if(order_count>=1,1,0)) buy3timeLastRatio ,
    date_format('2022-05-20' ,'yyyy-MM') stat_mn,
    '2022-05-20' stat_date
from
(
    select 
        tmp_order.user_id,
        tmp_sku.category1_id,
        tmp_sku.category1_name,
        tmp_sku.tm_id,
        sum(order_count) order_count
    from tmp_order
    join tmp_sku
    on tmp_order.sku_id=tmp_sku.id
    group by tmp_order.user_id,tmp_sku.category1_id,tmp_sku.category1_name,tmp_sku.tm_id
)tmp
group by tm_id, category1_id, category1_name;
```

3）查询导入结果

```sql
 select * from ads_sale_tm_category1_stat_mn;
```

## 五、地区主题

### 5.1 地区主题信息

1）建表语句

```typescript
create external table ads_area_topic(
    `dt` string COMMENT '统计日期',
    `id` bigint COMMENT '编号',
    `province_name` string COMMENT '省份名称',
    `area_code` string COMMENT '地区编码',
    `iso_code` string COMMENT 'iso编码',
    `region_id` string COMMENT '地区ID',
    `region_name` string COMMENT '地区名称',
    `login_day_count` bigint COMMENT '当天活跃设备数',
    `order_day_count` bigint COMMENT '当天下单次数',
    `order_day_amount` decimal(16,2) COMMENT '当天下单金额',
    `payment_day_count` bigint COMMENT '当天支付次数',
    `payment_day_amount` decimal(16,2) COMMENT '当天支付金额'
) COMMENT '地区主题信息'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_area_topic/';
```

2）导入数据

```sql
insert into table ads_area_topic
select
    '2022-05-20',
    id,
    province_name,
    area_code,
    iso_code,
    region_id,
    region_name,
    login_day_count,
    order_day_count,
    order_day_amount,
    payment_day_count,
    payment_day_amount
from dwt_area_topic;
```

3）查询导入结果

```sql
select * from ads_area_topic;
```

![img](https://img-blog.csdnimg.cn/71f92b8226c44249a373ea7a1276874e.png)





## 六、ADS层数据导入脚本

vim dwt_to_ads.sh

在脚本中填写如下内容：

```bash
#!/bin/bash
 
hive=/training/hive/bin/hive
APP=default
# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
    do_date=$1
else 
    do_date=`date -d "-1 day" +%F`
fi
 
sql="
set mapreduce.job.queuename=default;
insert into table ${APP}.ads_uv_count 
select  
    '$do_date' dt,
    daycount.ct,
    wkcount.ct,
    mncount.ct,
    if(date_add(next_day('$do_date','MO'),-1)='$do_date','Y','N') ,
    if(last_day('$do_date')='$do_date','Y','N') 
from 
(
    select  
        '$do_date' dt,
        count(*) ct
    from ${APP}.dwt_uv_topic
    where login_date_last='$do_date'  
)daycount join 
( 
    select  
        '$do_date' dt,
        count (*) ct
    from ${APP}.dwt_uv_topic
    where login_date_last>=date_add(next_day('$do_date','MO'),-7) 
    and login_date_last<= date_add(next_day('$do_date','MO'),-1) 
) wkcount on daycount.dt=wkcount.dt
join 
( 
    select  
        '$do_date' dt,
        count (*) ct
    from ${APP}.dwt_uv_topic
    where date_format(login_date_last,'yyyy-MM')=date_format('$do_date','yyyy-MM')  
)mncount on daycount.dt=mncount.dt;
insert into table ${APP}.ads_new_mid_count 
select
    login_date_first,
    count(*)
from ${APP}.dwt_uv_topic
where login_date_first='$do_date'
group by login_date_first;
insert into table ${APP}.ads_silent_count
select
    '$do_date',
    count(*) 
from ${APP}.dwt_uv_topic
where login_date_first=login_date_last
and login_date_last<=date_add('$do_date',-7);
insert into table ${APP}.ads_back_count
select
'$do_date',
concat(date_add(next_day('$do_date','MO'),-7),'_', date_add(next_day('$do_date','MO'),-1)),
    count(*)
from
(
    select
        mid_id
    from ${APP}.dwt_uv_topic
    where login_date_last>=date_add(next_day('$do_date','MO'),-7) 
    and login_date_last<= date_add(next_day('$do_date','MO'),-1)
    and login_date_first<date_add(next_day('$do_date','MO'),-7)
)current_wk
left join
(
    select
        mid_id
    from ${APP}.dws_uv_detail_daycount
    where dt>=date_add(next_day('$do_date','MO'),-7*2) 
    and dt<= date_add(next_day('$do_date','MO'),-7-1) 
    group by mid_id
)last_wk
on current_wk.mid_id=last_wk.mid_id
where last_wk.mid_id is null;
insert into table ${APP}.ads_wastage_count
select
     '$do_date',
     count(*)
from 
(
    select 
        mid_id
    from ${APP}.dwt_uv_topic
    where login_date_last<=date_add('$do_date',-7)
    group by mid_id
)t1;
insert into table ${APP}.ads_user_retention_day_rate
select
    '$do_date',--统计日期
    date_add('$do_date',-1),--新增日期
    1,--留存天数
    sum(if(login_date_first=date_add('$do_date',-1) and login_date_last='$do_date',1,0)),--$do_date的1日留存数
    sum(if(login_date_first=date_add('$do_date',-1),1,0)),--$do_date新增
    sum(if(login_date_first=date_add('$do_date',-1) and login_date_last='$do_date',1,0))/sum(if(login_date_first=date_add('$do_date',-1),1,0))*100
from ${APP}.dwt_uv_topic
union all
select
    '$do_date',--统计日期
    date_add('$do_date',-2),--新增日期
    2,--留存天数
    sum(if(login_date_first=date_add('$do_date',-2) and login_date_last='$do_date',1,0)),--$do_date的2日留存数
    sum(if(login_date_first=date_add('$do_date',-2),1,0)),--$do_date新增
    sum(if(login_date_first=date_add('$do_date',-2) and login_date_last='$do_date',1,0))/sum(if(login_date_first=date_add('$do_date',-2),1,0))*100
from ${APP}.dwt_uv_topic
union all
select
    '$do_date',--统计日期
    date_add('$do_date',-3),--新增日期
    3,--留存天数
    sum(if(login_date_first=date_add('$do_date',-3) and login_date_last='$do_date',1,0)),--$do_date的3日留存数
    sum(if(login_date_first=date_add('$do_date',-3),1,0)),--$do_date新增
    sum(if(login_date_first=date_add('$do_date',-3) and login_date_last='$do_date',1,0))/sum(if(login_date_first=date_add('$do_date',-3),1,0))*100
from ${APP}.dwt_uv_topic;
insert into table ${APP}.ads_continuity_wk_count
select
    '$do_date',
    concat(date_add(next_day('$do_date','MO'),-7*3),'_',date_add(next_day('$do_date','MO'),-1)),
    count(*)
from
(
    select
        mid_id
    from
    (
        select
            mid_id
        from ${APP}.dws_uv_detail_daycount
        where dt>=date_add(next_day('$do_date','monday'),-7)
        and dt<=date_add(next_day('$do_date','monday'),-1)
        group by mid_id
        union all
        select
            mid_id
        from ${APP}.dws_uv_detail_daycount
        where dt>=date_add(next_day('$do_date','monday'),-7*2)
        and dt<=date_add(next_day('$do_date','monday'),-7-1)
        group by mid_id
        union all
        select
            mid_id
        from ${APP}.dws_uv_detail_daycount
        where dt>=date_add(next_day('$do_date','monday'),-7*3)
        and dt<=date_add(next_day('$do_date','monday'),-7*2-1)
        group by mid_id
    )t1
    group by mid_id
    having count(*)=3
)t2;
insert into table ${APP}.ads_continuity_uv_count
select
    '$do_date',
    concat(date_add('$do_date',-6),'_','$do_date'),
    count(*)
from
(
    select mid_id
    from
    (
        select mid_id      
        from
        (
            select 
                mid_id,
                date_sub(dt,rank) date_dif
            from
            (
                select 
                    mid_id,
                    dt,
                    rank() over(partition by mid_id order by dt) rank
                from ${APP}.dws_uv_detail_daycount
                where dt>=date_add('$do_date',-6) and dt<='$do_date'
            )t1
        )t2 
        group by mid_id,date_dif
        having count(*)>=3
    )t3 
    group by mid_id
)t4;
insert into table ${APP}.ads_user_topic
select
    '$do_date',
    sum(if(login_date_last='$do_date',1,0)),
    sum(if(login_date_first='$do_date',1,0)),
    sum(if(payment_date_first='$do_date',1,0)),
    sum(if(payment_count>0,1,0)),
    count(*),
    sum(if(login_date_last='$do_date',1,0))/count(*),
    sum(if(payment_count>0,1,0))/count(*),
    sum(if(login_date_first='$do_date',1,0))/sum(if(login_date_last='$do_date',1,0))
from ${APP}.dwt_user_topic;
with
tmp_uv as
(
    select
        '$do_date' dt,
        sum(if(array_contains(pages,'home'),1,0)) home_count,
        sum(if(array_contains(pages,'good_detail'),1,0)) good_detail_count
    from
    (
        select
            mid_id,
            collect_set(page_id) pages
        from ${APP}.dwd_page_log
        where dt='$do_date'
        and page_id in ('home','good_detail')
        group by mid_id
    )tmp
),
tmp_cop as
(
    select 
        '$do_date' dt,
        sum(if(cart_count>0,1,0)) cart_count,
        sum(if(order_count>0,1,0)) order_count,
        sum(if(payment_count>0,1,0)) payment_count
    from ${APP}.dws_user_action_daycount
    where dt='$do_date'
)
insert into table ${APP}.ads_user_action_convert_day
select
    tmp_uv.dt,
    tmp_uv.home_count,
    tmp_uv.good_detail_count,
    tmp_uv.good_detail_count/tmp_uv.home_count*100,
    tmp_cop.cart_count,
    tmp_cop.cart_count/tmp_uv.good_detail_count*100,
    tmp_cop.order_count,
    tmp_cop.order_count/tmp_cop.cart_count*100,
    tmp_cop.payment_count,
    tmp_cop.payment_count/tmp_cop.order_count*100
from tmp_uv
join tmp_cop
on tmp_uv.dt=tmp_cop.dt;
insert into table ${APP}.ads_product_info
select
    '$do_date' dt,
    sku_num,
    spu_num
from
(
    select
        '$do_date' dt,
        count(*) sku_num
    from
        ${APP}.dwt_sku_topic
) tmp_sku_num
join
(
    select
        '$do_date' dt,
        count(*) spu_num
    from
    (
        select
            spu_id
        from
            ${APP}.dwt_sku_topic
        group by
            spu_id
    ) tmp_spu_id
) tmp_spu_num
on
    tmp_sku_num.dt=tmp_spu_num.dt;
insert into table ${APP}.ads_product_sale_topN
select
    '$do_date' dt,
    sku_id,
    payment_amount
from
    ${APP}.dws_sku_action_daycount
where
    dt='$do_date'
order by payment_amount desc
limit 10;
insert into table ${APP}.ads_product_favor_topN
select
    '$do_date' dt,
    sku_id,
    favor_count
from
    ${APP}.dws_sku_action_daycount
where
    dt='$do_date'
order by favor_count desc
limit 10;
insert into table ${APP}.ads_product_cart_topN
select
    '$do_date' dt,
    sku_id,
    cart_count
from
    ${APP}.dws_sku_action_daycount
where
    dt='$do_date'
order by cart_count desc
limit 10;
insert into table ${APP}.ads_product_refund_topN
select
    '$do_date',
    sku_id,
    refund_last_30d_count/payment_last_30d_count*100 refund_ratio
from ${APP}.dwt_sku_topic
order by refund_ratio desc
limit 10;
insert into table ${APP}.ads_appraise_bad_topN
select
    '$do_date' dt,
    sku_id,
appraise_bad_count/(appraise_good_count+appraise_mid_count+appraise_bad_count+appraise_default_count) appraise_bad_ratio
from
    ${APP}.dws_sku_action_daycount
where
    dt='$do_date'
order by appraise_bad_ratio desc
limit 10;
insert into table ${APP}.ads_order_daycount
select
    '$do_date',
    sum(order_count),
    sum(order_amount),
    sum(if(order_count>0,1,0))
from ${APP}.dws_user_action_daycount
where dt='$do_date';
insert into table ${APP}.ads_payment_daycount
select
    tmp_payment.dt,
    tmp_payment.payment_count,
    tmp_payment.payment_amount,
    tmp_payment.payment_user_count,
    tmp_skucount.payment_sku_count,
    tmp_time.payment_avg_time
from
(
    select
        '$do_date' dt,
        sum(payment_count) payment_count,
        sum(payment_amount) payment_amount,
        sum(if(payment_count>0,1,0)) payment_user_count
    from ${APP}.dws_user_action_daycount
    where dt='$do_date'
)tmp_payment
join
(
    select
        '$do_date' dt,
        sum(if(payment_count>0,1,0)) payment_sku_count 
    from ${APP}.dws_sku_action_daycount
    where dt='$do_date'
)tmp_skucount on tmp_payment.dt=tmp_skucount.dt
join
(
    select
        '$do_date' dt,
        sum(unix_timestamp(payment_time)-unix_timestamp(create_time))/count(*)/60 payment_avg_time
    from ${APP}.dwd_fact_order_info
    where dt='$do_date'
    and payment_time is not null
)tmp_time on tmp_payment.dt=tmp_time.dt;
with 
tmp_order as
(
    select
        user_id,
        order_stats_struct.sku_id sku_id,
        order_stats_struct.order_count order_count
    from ${APP}.dws_user_action_daycount lateral view explode(order_detail_stats) tmp as order_stats_struct
    where date_format(dt,'yyyy-MM')=date_format('$do_date','yyyy-MM')
),
tmp_sku as
(
    select
        id,
        tm_id,
        category1_id,
        category1_name
    from ${APP}.dwd_dim_sku_info
    where dt='$do_date'
)
insert into table ${APP}.ads_sale_tm_category1_stat_mn
select
    tm_id,
    category1_id,
    category1_name,
    sum(if(order_count>=1,1,0)) buycount,
    sum(if(order_count>=2,1,0)) buyTwiceLast,
    sum(if(order_count>=2,1,0))/sum( if(order_count>=1,1,0)) buyTwiceLastRatio,
    sum(if(order_count>=3,1,0))  buy3timeLast  ,
    sum(if(order_count>=3,1,0))/sum( if(order_count>=1,1,0)) buy3timeLastRatio ,
    date_format('$do_date' ,'yyyy-MM') stat_mn,
    '$do_date' stat_date
from
(
    select 
        tmp_order.user_id,
        tmp_sku.category1_id,
        tmp_sku.category1_name,
        tmp_sku.tm_id,
        sum(order_count) order_count
    from tmp_order
    join tmp_sku
    on tmp_order.sku_id=tmp_sku.id
    group by tmp_order.user_id,tmp_sku.category1_id,tmp_sku.category1_name,tmp_sku.tm_id
)tmp
group by tm_id, category1_id, category1_name;
insert into table ${APP}.ads_area_topic
select
    '$do_date',
    id,
    province_name,
    area_code,
    iso_code,
    region_id,
    region_name,
    login_day_count,
    order_day_count,
    order_day_amount,
    payment_day_count,
    payment_day_amount
from ${APP}.dwt_area_topic;
"
 
$hive -e "$sql"
```

2）增加脚本执行权限 chmod 777 dwt_to_ads.sh

3）执行脚本导入数据 dwt_to_ads.sh 2022-05-21

4）查看导入数据

```vbnet
select * from ads_uv_count;
select * from ads_new_mid_count;
select * from ads_silent_count;
select * from ads_back_count';
select * from ads_wastage_count;
select * from ads_user_retention_day_rate;
select * from ads_continuity_wk_count';
select * from ads_continuity_uv_count;
select * from ads_user_topic where dt='2020-06-15';
select * from ads_user_action_convert_day;
select * from ads_product_info;
select * from ads_product_sale_topN;
select * from ads_product_favor_topN;
select * from ads_product_cart_topN;
select * from ads_product_refund_topN;
select * from ads_appraise_bad_topN;
select * from ads_order_daycount;
select * from ads_payment_daycount;
select * from ads_sale_tm_category1_stat_mn;
select * from ads_area_topic';
```

完成！！！