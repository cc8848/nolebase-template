- [HIve数仓新零售项目DWS层的构建(Grouping sets)模型_陈万君Allen的博客-CSDN博客](https://blog.csdn.net/weixin_53280379/article/details/127953200)

# 前言

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd88bf2eb9e44b61a027ac7fc01e95ef.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6678905cbee74b46b6d3eb186d5e063a.png)
这是一个线下真实HIve数仓的一个搭建项目，还是比较复杂的，主要和大家一起分享一下整个HIve数仓的思路。
整个项目分为：
1.ODS层
2.DWD层
3.DWB层
4.DWS层
5.DM层
6.RPT层
每一层都有每一层的知识点。我会和大家分享从数据源MySQL开始，如何搭建整个完整的项目。

------

# 一、Grouping sets 模型介绍

一种高效的替代多个UNION ALL语法的模型，个人比较喜欢，非常灵活，速度快。

以下是一个demo案例。

需求：

分别按照月（month）、天（day）、月和天（month,day）统计**来访用户userid个数**，并获取三者的结果集（一起插入到目标宽表中）。

```c
create table test.t_user(
    month string, 
    day string, 
    userid string
) 
row format delimited fields terminated by ',';

--数据样例
2015-03,2015-03-10,user1
2015-03,2015-03-10,user5
2015-03,2015-03-12,user7
2015-04,2015-04-12,user3
2015-04,2015-04-13,user2
2015-04,2015-04-13,user4
2015-04,2015-04-16,user4
2015-03,2015-03-10,user2
2015-03,2015-03-10,user3
2015-04,2015-04-12,user5
2015-04,2015-04-13,user6
2015-04,2015-04-15,user3
2015-04,2015-04-15,user2
2015-04,2015-04-16,user1
```

- UNION ALL 写法 ，比较丑而且速度慢，效率低

```c
--3个分组统计而已，简单。统计完再使用union all合并结果集。
--注意union all合并结果集需要各个查询返回字段个数、类型一致，因此需要合理的使用null来填充返回结果。
select month,
       null,
       count(userid)
from test.t_user
group by month

union all

select null,
       day,
       count(userid)
from test.t_user
group by day

union all

select month,
       day,
       count(userid)
from test.t_user
group by month,day;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/d5a0702454004c24a6c332c2d0c1404c.png)

- grouping sets模型写法
- 根据不同的维度组合进行聚合，等价于将不同维度的GROUP BY结果集进行UNION ALL。

```c
-- Hive的写法
select 
    month,day,count(userid) 
from test.t_user 
    group by month,day 
grouping sets (month,day,(month,day));

-- presto的写法
select
month,
day,
count(*) as cnt
from  test.t_user
group by
grouping sets (month,day,(month,day))
```

- grouping介绍与使用
- 功能：使用grouping操作来判断当前数据是按照哪个字段来分组的。
- 对于给定的分组，如果分组中包含相应的列，则将位设置为0，否则将其设置为1

```c
select month,
       day,
       count(userid),
       grouping(month)      as m,
       grouping(day)        as d,
       grouping(month, day) as m_d
from test.t_user
group by
   grouping sets (month, day, (month, day));
```

# 二、DWS层功能与职责

DWS层: 基于主题统计分析, 此层一般是用于最细粒度的统计操作

- 维度组合:

  日期
  日期+城市
  日期+城市+商圈
  日期+城市+商圈+店铺
  日期+品牌
  日期+大类
  日期+大类+中类
  日期+大类+中列+小类

- 指标:
  销售收入、平台收入、配送成交额、小程序成交额、安卓APP成交额、苹果APP成交额、PC商城成交额、订单量、参 评单量、差评单量、配送单量、退款单量、小程序订单量、安卓APP订单量、苹果APP订单量、PC商城订单量。

# 三、销售主题统计宽表

最终要求通过group_type来判断指标来自哪个维度的聚合
![在这里插入图片描述](https://img-blog.csdnimg.cn/f370f8c33ae84bd09857ee420ca16dbc.png)

## 1.构建目标表

```c
drop database if exists yp_dws
create database if not exists yp_dws;

-- 销售主题日统计宽表
DROP TABLE IF EXISTS yp_dws.dws_sale_daycount;
CREATE TABLE yp_dws.dws_sale_daycount(
   city_id string COMMENT '城市id',
   city_name string COMMENT '城市name',
   trade_area_id string COMMENT '商圈id',
   trade_area_name string COMMENT '商圈名称',
   store_id string COMMENT '店铺的id',
   store_name string COMMENT '店铺名称',
   brand_id string COMMENT '品牌id',
   brand_name string COMMENT '品牌名称',
   max_class_id string COMMENT '商品大类id',
   max_class_name string COMMENT '大类名称',
   mid_class_id string COMMENT '中类id',
   mid_class_name string COMMENT '中类名称',
   min_class_id string COMMENT '小类id',
   min_class_name string COMMENT '小类名称',

   -- 经验字段: 用于标记每一条数据是按照哪个维度计算出来的
   group_type string COMMENT '分组类型：store，trade_area，city，brand，min_class，mid_class，max_class，all',

   --   =======日统计=======
   --   销售收入
   sale_amt DECIMAL(38,2) COMMENT '销售收入',
   --   平台收入
   plat_amt DECIMAL(38,2) COMMENT '平台收入',
   -- 配送成交额
   deliver_sale_amt DECIMAL(38,2) COMMENT '配送成交额',
   -- 小程序成交额
   mini_app_sale_amt DECIMAL(38,2) COMMENT '小程序成交额',
   -- 安卓APP成交额
   android_sale_amt DECIMAL(38,2) COMMENT '安卓APP成交额',
   --  苹果APP成交额
   ios_sale_amt DECIMAL(38,2) COMMENT '苹果APP成交额',
   -- PC商城成交额
   pcweb_sale_amt DECIMAL(38,2) COMMENT 'PC商城成交额',
   -- 成交单量
   order_cnt BIGINT COMMENT '成交单量',
   -- 参评单量
   eva_order_cnt BIGINT COMMENT '参评单量comment=>cmt',
   -- 差评单量
   bad_eva_order_cnt BIGINT COMMENT '差评单量negtive-comment=>ncmt',
   -- 配送成交单量
   deliver_order_cnt BIGINT COMMENT '配送单量',
   -- 退款单量
   refund_order_cnt BIGINT COMMENT '退款单量',
   -- 小程序成交单量
   miniapp_order_cnt BIGINT COMMENT '小程序成交单量',
   -- 安卓APP订单量
   android_order_cnt BIGINT COMMENT '安卓APP订单量',
   -- 苹果APP订单量
   ios_order_cnt BIGINT COMMENT '苹果APP订单量',
   -- PC商城成交单量
   pcweb_order_cnt BIGINT COMMENT 'PC商城成交单量'
)
COMMENT '销售主题日统计宽表'
PARTITIONED BY(dt STRING)
ROW format delimited fields terminated BY '\t'
stored AS orc tblproperties ('orc.compress' = 'SNAPPY');
```

## 2.Presto Grouping sets语法实现

```c
insert into yp_dws.dws_sale_daycount
with t0 as (
   select
     -- 列裁剪
     -- 维度字段
     od.dt,
     city_id,
     city_name,
     trade_area_id,
     trade_area_name,
     store_name,
     brand_id,
     brand_name,
     max_class_name,
     max_class_id,
     mid_class_name,
     mid_class_id,
     min_class_name,
     min_class_id,

     -- 指标字段
    order_id,
    order_amount,
    total_price,
    plat_fee,
    delivery_fee,
    order_from,
    evaluation_id,
    geval_scores,
    delievery_id,
    refund_id,
    od.store_id,
    row_number() over (partition by order_id,goods_id ) as rk1, -- 过滤脏数据
    row_number() over (partition by order_id ) as rk2
   from yp_dwb.dwb_order_detail od
     left join  yp_dwb.dwb_shop_detail  sd on od.store_id = sd.id
     left join  yp_dwb.dwb_goods_detail gd on od.goods_id = gd.id
 )
select
    city_id,
    city_name,
    trade_area_id,
    trade_area_name,
    store_id,
    store_name,
    brand_id,
    brand_name,
    max_class_id,
    max_class_name,
    mid_class_id,
    mid_class_name,
    min_class_id,
    min_class_name,
    case when grouping(store_id) = 0      -- if
           then 'store'  -- 日期 + 城市 + 商圈 + 店铺
         when grouping(trade_area_id) = 0  -- else if
           then 'trade_area'  --日期 + 城市 + 商圈
         when grouping(city_id) = 0   -- else if
            then 'city '      --日期 + 城市
         when grouping(brand_id) = 0  -- else if
           then 'brand'  -- 日期 = 品牌
         when grouping(min_class_id) = 0  -- else if
           then 'min_class'  -- 日期 + 大类 + 中类 + 小类
         when grouping(mid_class_id) = 0   -- else if
           then 'mid_class'  -- 日期 + 大类 + 中类
         when grouping(max_class_id) = 0
           then 'max_clas'  -- 日期 + 大类
         else
           'all'  -- 日期
    end as group_type,
      -- 总销售额
    case when grouping(store_id) = 0
            then sum(if(store_id is not null,total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null,total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null,total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null,total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null,total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null,total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null,total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null,total_price,0))
        else
            sum(if(dt is not null,total_price,0))  -- 日期
    end as sale_amt,
    -- 平台收入
       case when grouping(store_id) = 0
            then sum(if(store_id is not null,plat_fee,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null,plat_fee,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null,plat_fee,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null,plat_fee,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null,plat_fee,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null,plat_fee,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null,plat_fee,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null,plat_fee,0))
        else
            sum(if(dt is not null,plat_fee,0))  -- 日期
    end as plat_amt,
     -- 配送成交额
       case when grouping(store_id) = 0
            then sum(if(store_id is not null and delievery_id is not null,total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null  and delievery_id is not null,total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null  and delievery_id is not null,total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null  and delievery_id is not null,total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null  and delievery_id is not null,total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and delievery_id is not null,total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and delievery_id is not null,total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null  and delievery_id is not null,total_price,0))
        else
            sum(if(dt is not null  and delievery_id is not null,total_price,0))  -- 日期
    end as deliver_sale_amt,
     -- 小程序成交额
       case when grouping(store_id) = 0
            then sum(if(store_id is not null and order_from = 'miniapp',total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null and order_from = 'miniapp',total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null  and order_from = 'miniapp',total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null  and order_from = 'miniapp',total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null  and order_from = 'miniapp',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'miniapp',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'miniapp',total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null  and order_from = 'miniapp',total_price,0))
        else
            sum(if(dt is not null  and order_from = 'miniapp',total_price,0))  -- 日期
    end as mini_app_sale_amt,
     -- android成交额
       case when grouping(store_id) = 0
            then sum(if(store_id is not null and order_from = 'android',total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null and order_from = 'android',total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null  and order_from = 'android',total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null  and order_from = 'android',total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null  and order_from = 'android',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'android',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'android',total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null  and order_from = 'android',total_price,0))
        else
            sum(if(dt is not null  and order_from = 'android',total_price,0))  -- 日期
    end as android_sale_amt,
      -- ios成交额
    case when grouping(store_id) = 0
            then sum(if(store_id is not null and order_from = 'ios',total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null and order_from = 'ios',total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null  and order_from = 'ios',total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null  and order_from = 'ios',total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null  and order_from = 'ios',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'ios',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'ios',total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null  and order_from = 'ios',total_price,0))
        else
            sum(if(dt is not null  and order_from = 'ios',total_price,0))  -- 日期
    end as ios_sale_amt,
       -- pcweb成交额
    case when grouping(store_id) = 0
            then sum(if(store_id is not null and order_from = 'pcweb',total_price,0))
         when grouping(trade_area_id) = 0
            then sum(if(trade_area_id is not null and order_from = 'pcweb',total_price,0))
         when  grouping(city_id) = 0
            then sum(if(city_id is not null  and order_from = 'pcweb',total_price,0))
         when grouping(brand_id) = 0
             then sum(if(brand_id is not null  and order_from = 'pcweb',total_price,0))
         when grouping(min_class_id) = 0
            then sum(if(min_class_id is not null  and order_from = 'pcweb',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'pcweb',total_price,0))
        when grouping(mid_class_id) = 0
             then sum(if(mid_class_id is not null  and order_from = 'pcweb',total_price,0))   -- 聚合定制
        when grouping(max_class_id) = 0
             then sum(if(max_class_id is not null  and order_from = 'pcweb',total_price,0))
        else
            sum(if(dt is not null  and order_from = 'pcweb',total_price,0))  -- 日期
    end as pcweb_sale_amt,
    -- 成交单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2 = 1,order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2 = 1,order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1,order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1,order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1,order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1,order_id,null))
        else
            count(if(dt is not null and rk2=1,order_id,null))  -- 日期
    end as order_cnt,
     -- 参评单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and evaluation_id is not null and evaluation_id is not null,order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and evaluation_id is not null,order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1 and evaluation_id is not null,order_id,null))
        else
            count(if(dt is not null and rk2=1 and evaluation_id is not null,order_id,null))  -- 日期
    end as eva_order_cnt,

     -- 差评单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))
        else
            count(if(dt is not null and rk2=1 and evaluation_id is not null and geval_scores <= 6,order_id,null))  -- 日期
    end as bad_eva_order_cnt,
    -- 配送单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and delievery_id is not null,order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1 and delievery_id is not null,order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1 and delievery_id is not null,order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1 and delievery_id is not null,order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1 and delievery_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and delievery_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and delievery_id is not null,order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1 and delievery_id is not null,order_id,null))
        else
            count(if(dt is not null and rk2=1 and delievery_id is not null,order_id,null))  -- 日期
    end as deliver_order_cnt,
    -- 退款单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and refund_id is not null,order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1 and refund_id is not null,order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1 and refund_id is not null,order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1 and refund_id is not null,order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1 and refund_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and refund_id is not null,order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and refund_id is not null,order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1 and refund_id is not null,order_id,null))
        else
            count(if(dt is not null and rk2=1 and refund_id is not null,order_id,null))  -- 日期
    end as refund_order_cnt,
    -- 小程序成交单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and order_from = 'miniapp',order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1 and order_from = 'miniapp',order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1  and order_from = 'miniapp',order_id,null))
        else
            count(if(dt is not null  and rk2=1 and order_from = 'miniapp',order_id,null))  -- 日期
    end as miniapp_order_cnt,
       -- android成交单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1 and order_from = 'android',order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not  null and rk2=1 and order_from = 'android',order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1 and order_from = 'android',order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1 and order_from = 'android',order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1  and order_from = 'android',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and order_from = 'android',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1 and order_from = 'android',order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1  and order_from = 'android',order_id,null))
        else
            count(if(dt is not null and rk2=1 and order_from = 'android',order_id,null))  -- 日期
    end as android_order_cnt,
    -- ios成交单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1  and order_from = 'ios',order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1  and order_from = 'ios',order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1   and order_from = 'ios',order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1   and order_from = 'ios',order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1   and order_from = 'ios',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1   and order_from = 'ios',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1   and order_from = 'ios',order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1   and order_from = 'ios',order_id,null))
        else
            count(if(dt is not null and rk2=1   and order_from = 'ios',order_id,null))  -- 日期
    end as ios_order_cnt,
      -- pcweb成交单量
    case when grouping(store_id) = 0
            then count(if(store_id is not null and rk2=1  and order_from = 'pcweb',order_id,null))
         when grouping(trade_area_id) = 0
            then count(if(trade_area_id is not null and rk2=1  and order_from = 'pcweb',order_id,null))
         when  grouping(city_id) = 0
            then count(if(city_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))
         when grouping(brand_id) = 0
             then count(if(brand_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))
         when grouping(min_class_id) = 0
            then count(if(min_class_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))
        when grouping(mid_class_id) = 0
             then count(if(mid_class_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))   -- 聚合定制
        when grouping(max_class_id) = 0
             then count(if(max_class_id is not null and rk2=1   and order_from = 'pcweb',order_id,null))
        else
            count(if(dt is not null  and order_from = 'pcweb',order_id,null))  -- 日期
    end as pcweb_order_cnt,
    dt
from t0
where rk1 = 1
group by
grouping sets (
 dt,
 (dt,city_id,city_name),
 (dt,city_id,city_name,trade_area_id,trade_area_name),
 (dt,city_id,city_name,trade_area_id,trade_area_name,store_id,store_name),
 (dt,brand_id,brand_name),
 (dt,max_class_id,max_class_name),
 (dt,max_class_id,max_class_name,mid_class_id,mid_class_name),
 (dt,max_class_id,max_class_name,mid_class_id,mid_class_name,min_class_id,min_class_name)
);
```

这里主要是运用了grouping和grouping sets的语法，如果不了解可以百度一下。

------

# 总结

这里介绍了HIve数仓新零售项目DWS层的构建(Grouping sets)模型，Grouping sets模型适合于多维度，多指标的稀疏宽表的构建，可以把不同的维度放在同一张宽表中，方便以后查询。同时在建立聚合字段的时候，可以根据每个维度进行定制聚合的操作。比较灵活。

如果对grouping和grouping sets的语法有疑问，可以留言讨论。