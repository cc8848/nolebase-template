- [HIve数仓新零售项目DWS层的构建(Full join)模型_陈万君Allen的博客-CSDN博客](https://blog.csdn.net/weixin_53280379/article/details/127954111)

# 前言

![在这里插入图片描述](https://img-blog.csdnimg.cn/a800e1ad438e49f5b3aa63ec5e901522.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/744a5170aa164752946a16dcdcddb521.png)

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

# 一、Full join模型思路

- 先统计各自结果小表数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/d4cb80909b15408fa68891489abca055.png)

- 将结果小表数据进行full join

![在这里插入图片描述](https://img-blog.csdnimg.cn/40ef4244c3f2499297831ee99294c829.png)

- 通过coalesce(t1.dt,t2.dt,t3.dt) 从full join的结果表中进行数据的抽取

![在这里插入图片描述](https://img-blog.csdnimg.cn/3cf3989f93ca46b489e908697c8cc49a.png)

- 去重，将日期和goods_id重复的数据去掉
  最后通过group by + sum 对数据进行去重即可
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/27e5b9a2bf154ef3a5d2189851f1bfc0.png)
  实现低维度，多指标的模型

# 二、商品主题

上次和大家介绍了Grouping sets 模型，主要适合于多维度，多指标的情况。今天通过Full join方法实现低维度，多指标的情况。

- 维度

日期（day）+商品

- 指标

下单次数、下单件数、下单金额、被支付次数、被支付件数、被支付金额、被退款次数、被退款件数、被退款金额、被加入购物车次数、被加入购物车件数、被收藏次数、好评数、中评数、差评数

## 1.确定目标表

```c
create table yp_dws.dws_sku_daycount 
(
    -- 维度字段
    dt STRING,
    sku_id string comment 'sku_id',
    sku_name string comment '商品名称',
    -- 指标
    order_count bigint comment '被下单次数',
    order_num bigint comment '被下单件数',
    order_amount decimal(38,2) comment '被下单金额',
    payment_count bigint  comment '被支付次数',
    payment_num bigint comment '被支付件数',
    payment_amount decimal(38,2) comment '被支付金额',
    refund_count bigint  comment '被退款次数',
    refund_num bigint comment '被退款件数',
    refund_amount  decimal(38,2) comment '被退款金额',
    cart_count bigint comment '被加入购物车次数',
    cart_num bigint comment '被加入购物车件数',
    favor_count bigint comment '被收藏次数',
    evaluation_good_count bigint comment '好评数',
    evaluation_mid_count bigint comment '中评数',
    evaluation_bad_count bigint comment '差评数'
) COMMENT '每日商品行为'
--PARTITIONED BY(dt STRING)
ROW format delimited fields terminated BY '\t'
stored AS orc tblproperties ('orc.compress' = 'SNAPPY');
```

## 2.实现代码

```c
with t0 as (
    select
        dt,
        goods_id as sku_id,
        goods_name as sku_name,
        order_id,
        buy_num,
        total_price,
        order_state,
        refund_id,
        evaluation_id,
        geval_scores,
        row_number() over (partition by order_id,goods_id) as rk
    from yp_dwb.dwb_order_detail
),
-- 被下单次数，被下单件数，被下单金额
t1 as (
    select
       dt,
       sku_id,
       sku_name,
       count(order_id) as order_count,  -- 被下单次数
       sum(buy_num) as order_num, -- 被下单件数
       sum(total_price) as order_amount -- 被下单金额
    from  t0
    where rk = 1
    group by dt,sku_id,sku_name
),
-- 被支付次数，被支付件数，被支付金额
t2 as (
    select
       dt,
       sku_id,
       sku_name,
       count(order_id) as payment_count,  -- 被支付次数
       sum(buy_num) as payment_num, -- 被支付件数
       sum(total_price) as payment_amount -- 被支付金额
    from  t0
    where rk = 1 and order_state not in (1,7)
    group by dt,sku_id,sku_name
),
-- 被退款次数，被退款件数，被退款金额
t3 as(
    select
       dt,
       sku_id,
       sku_name,
       count(order_id) as refund_count,  -- 被退款次数
       sum(buy_num) as refund_num, -- 被退款件数
       sum(total_price) as refund_amount -- 被退款金额
    from  t0
    where rk = 1 and refund_id is not null
    group by dt,sku_id,sku_name
),
-- 被加入购物车次数，被加入购物车件数
-- 通过分析我们发现，之前的DWB层中并没有关联和购物车相关联的表，我们需要去跨层去DWD层去访问数据
-- 通过分析发现fact_shop_cart表没有商品名，所以需要和dim_goods商品表进行关联，获取商品名
t4 as (
    select
      substring(sc.create_time,1,10) as dt,
      sc.goods_id as sku_id,
      g.goods_name as sku_name,
      count(sc.id) as cart_count, -- 被加入购物车次数
      sum(sc.buy_num) as cart_num -- 被加入购物车件数
    from yp_dwd.fact_shop_cart sc left join yp_dwd.dim_goods g on sc.goods_id = g.id
    where sc.end_date = '9999-99-99'
    group by substring(sc.create_time,1,10),goods_id,goods_name
),
-- 被收藏次数
t5 as (
    select
        substring(gc.create_time,1,10) as dt,
        gc.goods_id as sku_id,
        g.goods_name as sku_name,
        count(gc.id) as favor_count -- 被收藏次数
    from yp_dwd.fact_goods_collect gc left join yp_dwd.dim_goods g on gc.goods_id = g.id
    where gc.end_date = '9999-99-99'
    group by substring(gc.create_time,1,10),gc.goods_id,g.goods_name
),
-- 好评数 中评数，差评数
t6 as (
    select
        dt,
        sku_id,
        sku_name,
        count(if(geval_scores>=9,evaluation_id,null)) as evaluation_good_count, -- 好评数
        count(if(geval_scores<9 and geval_scores > 6,evaluation_id,null)) as evaluation_mid_count, -- 中评数
        count(if(geval_scores <= 6,evaluation_id,null)) as evaluation_bad_count -- 差评数
    from t0
    where rk = 1 and evaluation_id is not null
    group by dt,sku_id,sku_name
),
t7 as (
select
    coalesce(t1.dt,t2.dt,t3.dt,t4.dt,t5.dt,t6.dt) as dt,
    coalesce(t1.sku_id,t2.sku_id,t3.sku_id,t4.sku_id,t5.sku_id,t6.sku_id) as sku_id,
    coalesce(t1.sku_name,t2.sku_name,t3.sku_name,t4.sku_name,t5.sku_name,t6.sku_name) as sku_name,
    coalesce(t1.order_count,0) as order_count,
    coalesce(t1.order_num,0) as order_num,
    coalesce(t1.order_amount,0) as order_amount,
    coalesce(t2.payment_count,0) as payment_count,
    coalesce(t2.payment_num,0) as payment_num,
    coalesce(t2.payment_amount,0) as payment_amount,
    coalesce(t3.refund_count,0) as refund_count,
    coalesce(t3.refund_num,0) as refund_num,
    coalesce(t3.refund_amount,0) as refund_amount,
    coalesce(t4.cart_count,0) as cart_count,
    coalesce(t4.cart_num,0) as cart_num,
    coalesce(t5.favor_count,0) as favor_count,
    coalesce(t6.evaluation_good_count,0) as evaluation_good_count,
    coalesce(t6.evaluation_mid_count,0) as evaluation_mid_count,
    coalesce(t6.evaluation_bad_count,0) as evaluation_bad_count
from t1
    full join t2 on t1.dt = t2.dt and t1.sku_id = t2.sku_id
    full join t3 on t2.dt = t3.dt and t2.sku_id = t3.sku_id
    full join t4 on t3.dt = t4.dt and t3.sku_id = t4.sku_id
    full join t5 on t4.dt = t5.dt and t4.sku_id = t5.sku_id
    full join t6 on t5.dt = t6.dt and t5.sku_id = t6.sku_id
)
-- 对最终的结果进行分组去重
select
    dt,
    sku_id,
    sku_name,

    sum(order_count) as order_count,
    sum(order_num) as order_num,
    sum(order_amount) as order_amount,
    sum(payment_count) as payment_count,
    sum(payment_num) as payment_num,
    sum(payment_amount) as payment_amount,
    sum(refund_count) as refund_count,
    sum(refund_num) as refund_num,
    sum(refund_amount) as refund_amount,
    sum(cart_count) as cart_count,
    sum(cart_num) as cart_num,
    sum(favor_count) as favor_count,
    sum(evaluation_good_count) as evaluation_good_count,
    sum(evaluation_mid_count) as evaluation_mid_count,
    sum(evaluation_bad_count) as evaluation_bad_count
from t7
group by dt,sku_id,sku_name
```

------

# 总结

这里介绍了HIve数仓新零售项目DWS层的构建(Full join)模型，主要针对低维度，多指标的情况。Full join模型的主要思路为

1. 用with语句将dwb_order_detail表关键字段提取
2. 先统计6张结果小表数据
3. 将6张结果小表数据进行full join
4. 从full join的结果表中进行数据的抽取
5. 去重，将日期和goods_id重复的数据去掉