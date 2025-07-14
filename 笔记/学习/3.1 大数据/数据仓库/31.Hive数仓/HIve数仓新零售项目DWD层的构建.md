- [HIve数仓新零售项目DWD层的构建_陈万君Allen的博客-CSDN博客](https://blog.csdn.net/weixin_53280379/article/details/127950491)

# 前言

![在这里插入图片描述](https://img-blog.csdnimg.cn/2d5549353c0944899eae3bd82722d3d1.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/ae69991c956a4293a0b337234e8198ce.png)

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

# 一、DWD层功能与职责

- dwd层中文叫做明细数据层。

- 主要功能：

  - 数据清洗转换、提供质量保证；
  - 区分事实、维度。

- 表名规范

  dwd.fact_xxxxxx
  
  订单主副表、订单结算、订单组、订单退款、订单商品快照、购物车、店铺收藏等
  
  dwd.dim_yyyyyy
  
  用户、区域、时间、店铺、商圈、地址信息、商品、商品分类、品牌等

# 二、DWD层搭建–地域维度表–全量覆盖导入

```c
DROP TABLE if EXISTS yp_dwd.dim_district;
CREATE TABLE yp_dwd.dim_district(
  id string COMMENT '主键ID', 
  code string COMMENT '区域编码', 
  name string COMMENT '区域名称', 
  pid string COMMENT '父级ID', 
  alias string COMMENT '别名')
COMMENT '区域字典表'
row format delimited fields terminated by '\t'
stored as orc 
tblproperties ('orc.compress' = 'SNAPPY');
```

全量覆盖操作

```
INSERT overwrite TABLE yp_dwd.dim_district
select * from yp_ods.t_district
WHERE code IS NOT NULL AND name IS NOT NULL;
```

# 二、DWD层搭建–订单评价表–增量导入

```c
#解释：每一次增量的数据都创建一个分区进行报错
DROP TABLE if EXISTS yp_dwd.fact_goods_evaluation;
CREATE TABLE yp_dwd.fact_goods_evaluation(
  id string, 
  user_id string COMMENT '评论人id', 
  store_id string COMMENT '店铺id', 
  order_id string COMMENT '订单id', 
  geval_scores int COMMENT '综合评分', 
  geval_scores_speed int COMMENT '送货速度评分0-5分(配送评分)', 
  geval_scores_service int COMMENT '服务评分0-5分', 
  geval_isanony tinyint COMMENT '0-匿名评价，1-非匿名', 
  create_user string, 
  create_time string, 
  update_user string, 
  update_time string, 
  is_valid tinyint COMMENT '0 ：失效，1 ：开启')
COMMENT '订单评价表'
partitioned by (dt string)
row format delimited fields terminated by '\t'
stored as orc 
tblproperties ('orc.compress' = 'SNAPPY');
```

- 第一次导入（全量）

```c
-- 从ods层进行加载
INSERT overwrite TABLE yp_dwd.fact_goods_evaluation PARTITION(dt)
select 
   id,
   user_id,
   store_id,
   order_id,
   geval_scores,
   geval_scores_speed,
   geval_scores_service,
   geval_isanony,
   create_user,
   create_time,
   update_user,
   update_time,
   is_valid,
   substr(create_time, 1, 10) as dt  
from yp_ods.t_goods_evaluation;
```

- 增量导入操作

```c
INSERT into TABLE yp_dwd.fact_goods_evaluation PARTITION(dt)
select 
   id,
   user_id,
   store_id,
   order_id,
   geval_scores,
   geval_scores_speed,
   geval_scores_service,
   geval_isanony,
   create_user,
   create_time,
   update_user,
   update_time,
   is_valid,
   substr(create_time, 1, 10) as dt
from yp_ods.t_goods_evaluation
where dt='2022-11-19';
```

# 三、DWD层搭建–订单事实表–循环与拉链导入

拉链表是面试重点，如果面数仓相关岗位，面试官特别爱问。

```c
DROP TABLE if EXISTS yp_dwd.fact_shop_order;
CREATE TABLE if not exists yp_dwd.fact_shop_order(  -- 拉链表
  id string COMMENT '根据一定规则生成的订单编号',
  order_num string COMMENT '订单序号',
  buyer_id string COMMENT '买家的userId',
  store_id string COMMENT '店铺的id',
  order_from string COMMENT '此字段可以转换 1.安卓\; 2.ios\; 3.小程序H5 \; 4.PC',
  order_state int COMMENT '订单状态:1.已下单\; 2.已付款, 3. 已确认 \;4.配送\; 5.已完成\; 6.退款\;7.已取消',
  create_date string COMMENT '下单时间',
  finnshed_time timestamp COMMENT '订单完成时间,当配送员点击确认送达时,进行更新订单完成时间,后期需要根据订单完成时间,进行自动收货以及自动评价',
  is_settlement tinyint COMMENT '是否结算\;0.待结算订单\; 1.已结算订单\;',
  is_delete tinyint COMMENT '订单评价的状态:0.未删除\;  1.已删除\;(默认0)',
  evaluation_state tinyint COMMENT '订单评价的状态:0.未评价\;  1.已评价\;(默认0)',
  way string COMMENT '取货方式:SELF自提\;SHOP店铺负责配送',
  is_stock_up int COMMENT '是否需要备货 0：不需要    1：需要    2:平台确认备货  3:已完成备货 4平台已经将货物送至店铺 ',
  create_user string,
  create_time string,
  update_user string,
  update_time string,
  is_valid tinyint COMMENT '是否有效  0: false\; 1: true\;   订单是否有效的标志',
  end_date string COMMENT '拉链结束日期'
) COMMENT '订单表'
partitioned by (start_date string)
row format delimited fields terminated by '\t'
stored as orc tblproperties ('orc.compress' = 'SNAPPY');
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/4ea4445d204e4775a2b374834b0592a7.png)

- 首次导入
- - 如果是动态分区插入，别忘了相关参数
- 如果ods层中表的字段有枚举类型，可以在ETL到dwd的过程中使用case when语句转换。

```c
INSERT overwrite TABLE yp_dwd.fact_shop_order PARTITION (start_date)
SELECT 
   id,
   order_num,
   buyer_id,
   store_id,
   case order_from 
      when 1
      then 'android'
      when 2
      then 'ios'
      when 3
      then 'miniapp'
      when 4
      then 'pcweb'
      else 'other'
      end
      as order_from,
   order_state,
   create_date,
   finnshed_time,
   is_settlement,
   is_delete,
   evaluation_state,
   way,
   is_stock_up,
   create_user,
   create_time,
   update_user,
   update_time,
   is_valid,
   '9999-99-99' end_date,
    dt as start_date
FROM yp_ods.t_shop_order;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/d340eb75516f47b193808e1d47b73ba6.png)

- 拉链操作

```c
insert overwrite table yp_dwd.fact_shop_order partition (start_date)
select *
from (
   --1、ods表的新分区数据(有新增和更新的数据)
         select id,
                order_num,
                buyer_id,
                store_id,
                case order_from
                    when 1
                        then 'android'
                    when 2
                        then 'ios'
                    when 3
                        then 'miniapp'
                    when 4
                        then 'pcweb'
                    else 'other'
                    end
                    as order_from,
                order_state,
                create_date,
                finnshed_time,
                is_settlement,
                is_delete,
                evaluation_state,
                way,
                is_stock_up,
                create_user,
                create_time,
                update_user,
                update_time,
                is_valid,
                '9999-99-99' end_date,
               '2022-11-19' as start_date
         from yp_ods.t_shop_order
         where dt='2022-11-19'

         union all

    -- 2、历史拉链表数据，并根据up_id判断更新end_time有效期
         select
             fso.id,
             fso.order_num,
             fso.buyer_id,
             fso.store_id,
             fso.order_from,
             fso.order_state,
             fso.create_date,
             fso.finnshed_time,
             fso.is_settlement,
             fso.is_delete,
             fso.evaluation_state,
             fso.way,
             fso.is_stock_up,
             fso.create_user,
             fso.create_time,
             fso.update_user,
             fso.update_time,
             fso.is_valid,
             --3、更新end_time：如果没有匹配到变更数据，或者当前已经是无效的历史数据，则保留原始end_time过期时间；否则变更end_time时间为前天（昨天之前有效）
             if (tso.id is not null and fso.end_date='9999-99-99',date_add(tso.dt, -1), fso.end_date) end_time,
             fso.start_date
         from yp_dwd.fact_shop_order fso left join (select * from yp_ods.t_shop_order where dt='2022-11-19') tso
         on fso.id=tso.id
     ) his
order by his.id, start_date;
```

------

# 总结

这里介绍了HIve数仓新零售项目DWD层的构建，主要三种方式
.

1. 全量覆盖导入
2. 增量导入
3. 循环与拉链导入