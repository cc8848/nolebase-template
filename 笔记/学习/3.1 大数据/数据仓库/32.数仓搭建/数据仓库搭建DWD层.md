- [数据仓库搭建DWD层_一个人的牛牛的博客-CSDN博客_数据仓库dwd层](https://blog.csdn.net/qq_55906442/article/details/124974677)

# 一、用户行为日志

## 1.1日志格式

1.页面埋点日志

![img](https://img-blog.csdnimg.cn/71e180016a034cc7b31116cf6e276c19.png)

 2.启动日志

![img](https://img-blog.csdnimg.cn/5653feae624e4e1ea2dd0db7d22621b4.png)



## 1.2get_json_object函数使用

1.取出第一个json的值

```lisp
select get_json_object('[{"name":"大郎","sex":"男","age":"25"},{"name":"西门庆","sex":"男","age":"47"}]','$[0]');
```

![img](https://img-blog.csdnimg.cn/45b76c5538a54434bd8181d13cf5fd9c.png)

2.取出第一个json的age字段的值

```lisp
SELECT get_json_object('[{"name":"大郎","sex":"男","age":"25"},{"name":"西门庆","sex":"男","age":"47"}]',"$[0].age");
```

 ![img](https://img-blog.csdnimg.cn/2d84435ba4f94e36bf40ace357fbbc0c.png)

## 1.3启动日志表

1.修改CombineHiveInputFormat为HiveInputFormat

```lua
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
```

2.启动日志解析思路：启动日志表中每行数据对应一个启动记录，一个启动记录应该包含日志中的公共信息和启动信息。先将所有包含start字段的日志过滤出来，然后使用[get_json_object](https://so.csdn.net/so/search?q=get_json_object&spm=1001.2101.3001.7020)函数解析每个字段。

![img](https://img-blog.csdnimg.cn/96b9b6bfd4034448a41d8dde23fbb543.png)

1）建表语句

```typescript
CREATE EXTERNAL TABLE dwd_start_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `entry` string COMMENT ' icon手机图标  notice 通知   install 安装后启动',
    `loading_time` bigint COMMENT '启动加载时间',
    `open_ad_id` string COMMENT '广告页ID ',
    `open_ad_ms` bigint COMMENT '广告总共播放时间', 
    `open_ad_skip_ms` bigint COMMENT '用户跳过广告时点', 
    `ts` bigint COMMENT '时间'
) COMMENT '启动日志表'
PARTITIONED BY (dt string) -- 按照时间创建分区
stored as parquet -- 采用parquet列式存储
LOCATION '/warehouse/gmall/dwd/dwd_start_log' -- 指定在HDFS上存储位置
TBLPROPERTIES('parquet.compression'='lzo') -- 采用LZO压缩
;
```

 \#说明：数据采用parquet存储方式，是可以支持切片的，不需要再对数据创建索引。如果单纯的text方式存储数据，需要采用支持切片的，lzop压缩方式并创建索引。

2）数据导入

```sql
insert overwrite table dwd_start_log partition(dt='2022-05-20')
select 
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.ts')
from ods_log
where dt='2022-05-20'
and get_json_object(line,'$.start') is not null;
```

3）查看数据

```sql
select * from dwd_start_log where dt='2022-05-20' limit 2;
```

## 1.4页面日志表

页面日志解析思路：页面日志表中每行数据对应一个页面访问记录，一个页面访问记录应该包含日志中的公共信息和页面信息。先将所有包含page字段的日志过滤出来，然后使用get_json_object函数解析每个字段。

![img](https://img-blog.csdnimg.cn/7ff080dd3f734a3f872f8d75493f5f3c.png)

1. 建表

```typescript
CREATE EXTERNAL TABLE dwd_page_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `during_time` bigint COMMENT '持续时间毫秒',
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面ID ',
    `source_type` string COMMENT '来源类型', 
    `ts` bigint
) COMMENT '页面日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_page_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

2.数据导入

```sql
insert overwrite table dwd_page_log partition(dt='2020-05-20')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.ts')
from ods_log
where dt='2020-05-20'
and get_json_object(line,'$.page') is not null;
```

3.查看数据

```sql
select * from dwd_page_log where dt='2020-05-20' limit 2;
```

## 1.5动作日志表

动作日志解析思路：动作日志表中每行数据对应用户的一个动作记录，一个动作记录应当包含公共信息、页面信息以及动作信息。先将包含action字段的日志过滤出来，然后通过UDTF函数，将action数组“炸开”（类似于explode函数的效果），然后使用get_json_object函数解析每个字段。

![img](https://img-blog.csdnimg.cn/7eefcf7833444735b2eaa859bb6276ad.png)

 1.建表

```typescript
CREATE EXTERNAL TABLE dwd_action_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `during_time` bigint COMMENT '持续时间毫秒', 
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面id ',
    `source_type` string COMMENT '来源类型', 
    `action_id` string COMMENT '动作id',
    `item` string COMMENT '目标id ',
    `item_type` string COMMENT '目标类型', 
    `ts` bigint COMMENT '时间'
) COMMENT '动作日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_action_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

2.创建UDTF函数——设计思路

![img](https://img-blog.csdnimg.cn/2e5b644afe1a49a99f4a9cc6ce395a6b.png)

![img](https://img-blog.csdnimg.cn/467d5f46501a4402a0234b8953960c4c.png)

3.创建UDTF函数--编写代码

1）创建maven工程：hivefunction

2）创建包名：com.zj.hive.udtf

3）引入依赖

```xml
<dependencies>
    <!--添加hive依赖-->
    <dependency>
        <groupId>org.apache.hive</groupId>
        <artifactId>hive-exec</artifactId>
        <version>3.1.2</version>
    </dependency>
</dependencies>
```

4）编码

```java
package com.zj.hive.udtf;
 
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
import org.json.JSONArray;
 
import java.util.ArrayList;
import java.util.List;
 
public class ExplodeJSONArray extends GenericUDTF {
 
    @Override
    public StructObjectInspector initialize(StructObjectInspector argOIs) throws UDFArgumentException {
 
        // 1 参数合法性检查
        if (argOIs.getAllStructFieldRefs().size() != 1){
            throw new UDFArgumentException("ExplodeJSONArray 只需要一个参数");
        }
 
        // 2 第一个参数必须为string
        if(!"string".equals(argOIs.getAllStructFieldRefs().get(0).getFieldObjectInspector().getTypeName())){
            throw new UDFArgumentException("json_array_to_struct_array的第1个参数应为string类型");
        }
 
        // 3 定义返回值名称和类型
        List<String> fieldNames = new ArrayList<String>();
        List<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();
 
        fieldNames.add("items");
        fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
 
        return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames, fieldOIs);
    }
 
    public void process(Object[] objects) throws HiveException {
 
        // 1 获取传入的数据
        String jsonArray = objects[0].toString();
 
        // 2 将string转换为json数组
        JSONArray actions = new JSONArray(jsonArray);
        
        // 3 循环一次，取出数组中的一个json，并写出
        for (int i = 0; i < actions.length(); i++) {
 
            String[] result = new String[1];
            result[0] = actions.getString(i);
            forward(result);
        }
    }
 
    public void close() throws HiveException {
 
    }
}
```

5）创建函数

（1）打包。

（2）将hivefunction-1.0-SNAPSHOT.jar上传到hadoop01的/tools/，然后再将该jar包上传到HDFS的/user/hive/jars路径下。

```bash
hdfs dfs -mkdir -p /user/hive/jars
hdfs dfs -put hivefunction-1.0-SNAPSHOT.jar /user/hive/jars
```

（3）创建永久函数与开发好的java class关联

```sql
create function explode_json_array as 'com.zj.hive.udtf.ExplodeJSONArray' using jar 'hdfs://hadoop01:9000/user/hive/jars/hivefunction-1.0-SNAPSHOT.jar';
```

（4）注意：如果修改了自定义函数重新生成jar包怎么处理？只需要替换HDFS路径上的旧jar包，然后重启Hive客户端即可。

```sql
insert overwrite table dwd_action_log partition(dt='2022-05-20')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(action,'$.action_id'),
    get_json_object(action,'$.item'),
    get_json_object(action,'$.item_type'),
    get_json_object(action,'$.ts')
from ods_log lateral view explode_json_array(get_json_object(line,'$.actions')) tmp as action
where dt='2022-05-20'
and get_json_object(line,'$.actions') is not null;
```

4.查看数据

```sql
select * from dwd_action_log where dt='2022-05-20' limit 2;
```

## 1.6曝光日志表

曝光日志解析思路：曝光日志表中每行数据对应一个曝光记录，一个曝光记录应当包含公共信息、页面信息以及曝光信息。先将包含display字段的日志过滤出来，然后通过UDTF函数，将display数组“炸开”（类似于explode函数的效果），然后使用get_json_object函数解析每个字段。

![img](https://img-blog.csdnimg.cn/40435d9c7cce4cb2b9aad0b1b49fd176.png)

1. 建表

```typescript
CREATE EXTERNAL TABLE dwd_display_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `during_time` bigint COMMENT 'app版本号',
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面ID ',
    `source_type` string COMMENT '来源类型', 
    `ts` bigint COMMENT 'app版本号',
    `display_type` string COMMENT '曝光类型',
    `item` string COMMENT '曝光对象id ',
    `item_type` string COMMENT 'app版本号', 
    `order` bigint COMMENT '出现顺序'
) COMMENT '曝光日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_display_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

2.导入数据

```sql
insert overwrite table dwd_display_log partition(dt='2020-05-20')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.ts'),
    get_json_object(display,'$.displayType'),
    get_json_object(display,'$.item'),
    get_json_object(display,'$.item_type'),
    get_json_object(display,'$.order')
from ods_log lateral view explode_json_array(get_json_object(line,'$.displays')) tmp as display
where dt='2020-05-20'
and get_json_object(line,'$.displays') is not null;
```

3.查看数据

```sql
select * from dwd_display_log where dt='2020-05-20' limit 2;
```

## 1.7错误日志表

错误日志解析思路：错误日志表中每行数据对应一个错误记录，为方便定位错误，一个错误记录应当包含与之对应的公共信息、页面信息、曝光信息、动作信息、启动信息以及错误信息。先将包含err字段的日志过滤出来，然后使用get_json_object函数解析所有字段。

1. 建表

```typescript
CREATE EXTERNAL TABLE dwd_error_log(
    `area_code` string COMMENT '地区编码',
    `brand` string COMMENT '手机品牌', 
    `channel` string COMMENT '渠道', 
    `model` string COMMENT '手机型号', 
    `mid_id` string COMMENT '设备id', 
    `os` string COMMENT '操作系统', 
    `user_id` string COMMENT '会员id', 
    `version_code` string COMMENT 'app版本号', 
    `page_item` string COMMENT '目标id ', 
    `page_item_type` string COMMENT '目标类型', 
    `last_page_id` string COMMENT '上页类型', 
    `page_id` string COMMENT '页面ID ',
    `source_type` string COMMENT '来源类型', 
    `entry` string COMMENT ' icon手机图标  notice 通知 install 安装后启动',
    `loading_time` string COMMENT '启动加载时间',
    `open_ad_id` string COMMENT '广告页ID ',
    `open_ad_ms` string COMMENT '广告总共播放时间', 
    `open_ad_skip_ms` string COMMENT '用户跳过广告时点',
    `actions` string COMMENT '动作',
    `displays` string COMMENT '曝光',
    `ts` string COMMENT '时间',
    `error_code` string COMMENT '错误码',
    `msg` string COMMENT '错误信息'
) COMMENT '错误日志表'
PARTITIONED BY (dt string)
stored as parquet
LOCATION '/warehouse/gmall/dwd/dwd_error_log'
TBLPROPERTIES('parquet.compression'='lzo');
```

说明：此处为对动作数组和曝光数组做处理，如需分析错误与单个动作或曝光的关联，可先使用explode_json_array函数将数组“炸开”，再使用get_json_object\函数获取具体字段。

2.导入数据

```sql
insert overwrite table dwd_error_log partition(dt='2020-05-20')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.actions'),
    get_json_object(line,'$.displays'),
    get_json_object(line,'$.ts'),
    get_json_object(line,'$.err.error_code'),
    get_json_object(line,'$.err.msg')
from ods_log 
where dt='2020-05-20'
and get_json_object(line,'$.err') is not null;
```

3.查看数据

```sql
select * from dwd_error_log where dt='2020-05-20' limit 2;
```

## 1.8 DWD层用户行为数据加载脚本

vim ods_to_dwd_log.sh

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
set mapreduce.job.queuename=hive;
insert overwrite table ${APP}.dwd_start_log partition(dt='$do_date')
select 
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.ts')
from ${APP}.ods_log
where dt='$do_date'
and get_json_object(line,'$.start') is not null;
insert overwrite table ${APP}.dwd_action_log partition(dt='$do_date')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(action,'$.action_id'),
    get_json_object(action,'$.item'),
    get_json_object(action,'$.item_type'),
    get_json_object(action,'$.ts')
from ${APP}.ods_log lateral view ${APP}.explode_json_array(get_json_object(line,'$.actions')) tmp as action
where dt='$do_date'
and get_json_object(line,'$.actions') is not null;
insert overwrite table ${APP}.dwd_display_log partition(dt='$do_date')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.ts'),
    get_json_object(display,'$.displayType'),
    get_json_object(display,'$.item'),
    get_json_object(display,'$.item_type'),
    get_json_object(display,'$.order')
from ${APP}.ods_log lateral view ${APP}.explode_json_array(get_json_object(line,'$.displays')) tmp as display
where dt='$do_date'
and get_json_object(line,'$.displays') is not null;
insert overwrite table ${APP}.dwd_page_log partition(dt='$do_date')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.during_time'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.ts')
from ${APP}.ods_log
where dt='$do_date'
and get_json_object(line,'$.page') is not null;
insert overwrite table ${APP}.dwd_error_log partition(dt='$do_date')
select
    get_json_object(line,'$.common.ar'),
    get_json_object(line,'$.common.ba'),
    get_json_object(line,'$.common.ch'),
    get_json_object(line,'$.common.md'),
    get_json_object(line,'$.common.mid'),
    get_json_object(line,'$.common.os'),
    get_json_object(line,'$.common.uid'),
    get_json_object(line,'$.common.vc'),
    get_json_object(line,'$.page.item'),
    get_json_object(line,'$.page.item_type'),
    get_json_object(line,'$.page.last_page_id'),
    get_json_object(line,'$.page.page_id'),
    get_json_object(line,'$.page.sourceType'),
    get_json_object(line,'$.start.entry'),
    get_json_object(line,'$.start.loading_time'),
    get_json_object(line,'$.start.open_ad_id'),
    get_json_object(line,'$.start.open_ad_ms'),
    get_json_object(line,'$.start.open_ad_skip_ms'),
    get_json_object(line,'$.actions'),
    get_json_object(line,'$.displays'),
    get_json_object(line,'$.ts'),
    get_json_object(line,'$.err.error_code'),
    get_json_object(line,'$.err.msg')
from ${APP}.ods_log 
where dt='$do_date'
and get_json_object(line,'$.err') is not null;
"
 
$hive -e "$sql"
```

增加脚本执行权限： chmod 777 ods_to_dwd_log.sh

脚本使用：ods_to_dwd_log.sh 2022-05-20

查询导入结果：`select * from dwd_start_log where dt='2022-05-20' limit 2;`

脚本执行时间企业开发中一般在每日凌晨30分~1点

# 二、业务数据

业务数据方面DWD层的搭建主要注意点在于维度建模，减少后续大量Join操作。

![img](https://img-blog.csdnimg.cn/cfd1d3731a1640aa8eb81dde2e1d7184.png)

## 2.1商品维度表（全量）

商品维度表主要是将商品表SKU表、商品一级分类、商品二级分类、商品三级分类、商品品牌表和商品SPU表联接为商品表。

![img](https://img-blog.csdnimg.cn/d4f6e8fa23124075a765d989c67338de.png)



### 建表

```sql
CREATE EXTERNAL TABLE `dwd_dim_sku_info` (
    `id` string COMMENT '商品id',
    `spu_id` string COMMENT 'spuid',
    `price` decimal(16,2) COMMENT '商品价格',
    `sku_name` string COMMENT '商品名称',
    `sku_desc` string COMMENT '商品描述',
    `weight` decimal(16,2) COMMENT '重量',
    `tm_id` string COMMENT '品牌id',
    `tm_name` string COMMENT '品牌名称',
    `category3_id` string COMMENT '三级分类id',
    `category2_id` string COMMENT '二级分类id',
    `category1_id` string COMMENT '一级分类id',
    `category3_name` string COMMENT '三级分类名称',
    `category2_name` string COMMENT '二级分类名称',
    `category1_name` string COMMENT '一级分类名称',
    `spu_name` string COMMENT 'spu名称',
    `create_time` string COMMENT '创建时间'
) COMMENT '商品维度表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_sku_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_dim_sku_info partition(dt='2022-05-10')
select  
    sku.id,
    sku.spu_id,
    sku.price,
    sku.sku_name,
    sku.sku_desc,
    sku.weight,
    sku.tm_id,
    ob.tm_name,
    sku.category3_id,
    c2.id category2_id,
    c1.id category1_id,
    c3.name category3_name,
    c2.name category2_name,
    c1.name category1_name,
    spu.spu_name,
    sku.create_time
from
(
    select * from ods_sku_info where dt='2022-05-10'
)sku
join
(
    select * from ods_base_trademark where dt='2022-05-10'
)ob on sku.tm_id=ob.tm_id
join
(
    select * from ods_spu_info where dt='2022-05-10'
)spu on spu.id = sku.spu_id
join 
(
    select * from ods_base_category3 where dt='2022-05-10'
)c3 on sku.category3_id=c3.id
join 
(
    select * from ods_base_category2 where dt='2022-05-10'
)c2 on c3.category2_id=c2.id 
join 
(
    select * from ods_base_category1 where dt='2022-05-10'
)c1 on c2.category1_id=c1.id;
```

### 查看数据

```
select * from dwd_dim_sku_info where dt='2022-05-10' limit 2;
```

## 2.2优惠券维度表（全量）

把ODS层ods_coupon_info表数据导入到DWD层优惠卷维度表，在导入过程中可以做适当的清洗。

### 建表

```sql
create external table dwd_dim_coupon_info(
    `id` string COMMENT '购物券编号',
    `coupon_name` string COMMENT '购物券名称',
    `coupon_type` string COMMENT '购物券类型 1 现金券 2 折扣券 3 满减券 4 满件打折券',
    `condition_amount` decimal(16,2) COMMENT '满额数',
    `condition_num` bigint COMMENT '满件数',
    `activity_id` string COMMENT '活动编号',
    `benefit_amount` decimal(16,2) COMMENT '减金额',
    `benefit_discount` decimal(16,2) COMMENT '折扣',
    `create_time` string COMMENT '创建时间',
    `range_type` string COMMENT '范围类型 1、商品 2、品类 3、品牌',
    `spu_id` string COMMENT '商品id',
    `tm_id` string COMMENT '品牌id',
    `category3_id` string COMMENT '品类id',
    `limit_num` bigint COMMENT '最多领用次数',
    `operate_time`  string COMMENT '修改时间',
    `expire_time`  string COMMENT '过期时间'
) COMMENT '优惠券维度表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_coupon_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_dim_coupon_info partition(dt='2022-05-10')
select
    id,
    coupon_name,
    coupon_type,
    condition_amount,
    condition_num,
    activity_id,
    benefit_amount,
    benefit_discount,
    create_time,
    range_type,
    spu_id,
    tm_id,
    category3_id,
    limit_num,
    operate_time,
    expire_time
from ods_coupon_info
where dt='2022-05-10';
```

### 查询数据

```sql
select * from dwd_dim_coupon_info where dt='2022-05-10' limit 2;
```

## 2.3活动维度表（全量）

### 建表

```sql
create external table dwd_dim_activity_info(
    `id` string COMMENT '编号',
    `activity_name` string  COMMENT '活动名称',
    `activity_type` string  COMMENT '活动类型',
    `start_time` string  COMMENT '开始时间',
    `end_time` string  COMMENT '结束时间',
    `create_time` string  COMMENT '创建时间'
) COMMENT '活动信息表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_activity_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_dim_activity_info partition(dt='2022-05-10')
select
    id,
    activity_name,
    activity_type,
    start_time,
    end_time,
    create_time
from ods_activity_info 
where dt='2022-05-10';
```

### 查询数据

```sql
select * from dwd_dim_activity_info where dt='2022-05-10' limit 2;
```

## 2.4地区维度表（特殊）

### 建表

```sql
CREATE EXTERNAL TABLE `dwd_dim_base_province` (
    `id` string COMMENT 'id',
    `province_name` string COMMENT '省市名称',
    `area_code` string COMMENT '地区编码',
    `iso_code` string COMMENT 'ISO编码',
    `region_id` string COMMENT '地区id',
    `region_name` string COMMENT '地区名称'
) COMMENT '地区维度表'
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_base_province/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_dim_base_province
select 
    bp.id,
    bp.name,
    bp.area_code,
    bp.iso_code,
    bp.region_id,
    br.region_name
from 
(
    select * from ods_base_province
) bp
join 
(
    select * from ods_base_region 
) br
on bp.region_id = br.id;
```

### 查询数据

```sql
select * from dwd_dim_base_province limit 2;
```

## 2.5时间维度表（特殊）

1.建表

```typescript
CREATE EXTERNAL TABLE `dwd_dim_date_info`(
    `date_id` string COMMENT '日',
    `week_id` string COMMENT '周',
    `week_day` string COMMENT '周的第几天',
    `day` string COMMENT '每月的第几天',
    `month` string COMMENT '第几月',
    `quarter` string COMMENT '第几季度',
    `year` string COMMENT '年',
    `is_workday` string COMMENT '是否是周末',
    `holiday_id` string COMMENT '是否是节假日'
) COMMENT '时间维度表'
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_date_info/'
tblproperties ("parquet.compression"="lzo");
```

2.把date_info.txt文件上传到hadoop102的/opt/module/db_log/路径

注意：由于dwd_dim_date_info是列式存储+LZO压缩。直接将date_info.txt文件导入到目标表，并不会直接转换为列式存储+LZO压缩。我们需要创建一张普通的临时表dwd_dim_date_info_tmp，将date_info.txt加载到该临时表中。最后通过查询临时表数据，把查询到的数据插入到最终的目标表中。

3.建表

```typescript
CREATE EXTERNAL TABLE `dwd_dim_date_info_tmp`(
    `date_id` string COMMENT '日',
    `week_id` string COMMENT '周',
    `week_day` string COMMENT '周的第几天',
    `day` string COMMENT '每月的第几天',
    `month` string COMMENT '第几月',
    `quarter` string COMMENT '第几季度',
    `year` string COMMENT '年',
    `is_workday` string COMMENT '是否是周末',
    `holiday_id` string COMMENT '是否是节假日'
) COMMENT '时间临时表'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/dwd/dwd_dim_date_info_tmp/';
```

 4.将数据导入临时表

```sql
load data local inpath '/opt/module/db_log/date_info.txt' into table dwd_dim_date_info_tmp;
```

5.将数据导入正式表

```sql
insert overwrite table dwd_dim_date_info select * from dwd_dim_date_info_tmp;
```

6.查看数据

```sql
select * from dwd_dim_date_info;
```

## 2.6支付事实表（事务型事实表）

### 建表

```sql
create external table dwd_fact_payment_info (
    `id` string COMMENT 'id',
    `out_trade_no` string COMMENT '对外业务编号',
    `order_id` string COMMENT '订单编号',
    `user_id` string COMMENT '用户编号',
    `alipay_trade_no` string COMMENT '支付宝交易流水编号',
    `payment_amount`    decimal(16,2) COMMENT '支付金额',
    `subject`         string COMMENT '交易内容',
    `payment_type` string COMMENT '支付类型',
    `payment_time` string COMMENT '支付时间',
    `province_id` string COMMENT '省份ID'
) COMMENT '支付事实表表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_payment_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_fact_payment_info partition(dt='2022-05-10')
select
    pi.id,
    pi.out_trade_no,
    pi.order_id,
    pi.user_id,
    pi.alipay_trade_no,
    pi.total_amount,
    pi.subject,
    pi.payment_type,
    pi.payment_time,
    oi.province_id
from
(
    select * from ods_payment_info where dt='2022-05-10'
)pi
join
(
    select id, province_id from ods_order_info where dt='2022-05-10'
)oi
on pi.order_id = oi.id;
```

### 查询数据

```sql
select * from dwd_fact_payment_info where dt='2022-05-10' limit 2;
```

## 2.7退款事实表（事务型事实表）

把ODS层ods_order_refund_info表数据导入到DWD层退款事实表，在导入过程中可以做适当的清洗。

### 建表

```sql
create external table dwd_fact_order_refund_info(
    `id` string COMMENT '编号',
    `user_id` string COMMENT '用户ID',
    `order_id` string COMMENT '订单ID',
    `sku_id` string COMMENT '商品ID',
    `refund_type` string COMMENT '退款类型',
    `refund_num` bigint COMMENT '退款件数',
    `refund_amount` decimal(16,2) COMMENT '退款金额',
    `refund_reason_type` string COMMENT '退款原因类型',
    `create_time` string COMMENT '退款时间'
) COMMENT '退款事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_order_refund_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据到入

```sql
insert overwrite table dwd_fact_order_refund_info partition(dt='2022-05-10')
select
    id,
    user_id,
    order_id,
    sku_id,
    refund_type,
    refund_num,
    refund_amount,
    refund_reason_type,
    create_time
from ods_order_refund_info
where dt='2022-05-10';
```

### 查询数据

```sql
select * from dwd_fact_order_refund_info where dt='2022-05-10' limit 2;
```

## 2.8评价事实表（事务型事实表）

把ODS层ods_comment_info表数据导入到DWD层评价事实表，在导入过程中可以做适当的清洗。

### 建表

```sql
create external table dwd_fact_comment_info(
    `id` string COMMENT '编号',
    `user_id` string COMMENT '用户ID',
    `sku_id` string COMMENT '商品sku',
    `spu_id` string COMMENT '商品spu',
    `order_id` string COMMENT '订单ID',
    `appraise` string COMMENT '评价',
    `create_time` string COMMENT '评价时间'
) COMMENT '评价事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_comment_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_fact_comment_info partition(dt='2022-05-10')
select
    id,
    user_id,
    sku_id,
    spu_id,
    order_id,
    appraise,
    create_time
from ods_comment_info
where dt='2022-05-10';
```

### 查询数据

```sql
select * from dwd_fact_comment_info where dt='2022-05-10' limit 2;
```

## 2.9订单明细事实表（事务型事实表）

### 建表

```sql
create external table dwd_fact_order_detail (
    `id` string COMMENT '订单编号',
    `order_id` string COMMENT '订单号',
    `user_id` string COMMENT '用户id',
    `sku_id` string COMMENT 'sku商品id',
    `sku_name` string COMMENT '商品名称',
    `order_price` decimal(16,2) COMMENT '商品价格',
    `sku_num` bigint COMMENT '商品数量',
    `create_time` string COMMENT '创建时间',
    `province_id` string COMMENT '省份ID',
    `source_type` string COMMENT '来源类型',
    `source_id` string COMMENT '来源编号',
    `original_amount_d` decimal(20,2) COMMENT '原始价格分摊',
    `final_amount_d` decimal(20,2) COMMENT '购买价格分摊',
    `feight_fee_d` decimal(20,2) COMMENT '分摊运费',
    `benefit_reduce_amount_d` decimal(20,2) COMMENT '分摊优惠'
) COMMENT '订单明细事实表表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_order_detail/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_fact_order_detail partition(dt='2022-05-10')
select
    id,
    order_id,
    user_id,
    sku_id,
    sku_name,
    order_price,
    sku_num,
    create_time,
    province_id,
    source_type,
    source_id,
    original_amount_d,
    if(rn=1,final_total_amount -(sum_div_final_amount - final_amount_d),final_amount_d),
    if(rn=1,feight_fee - (sum_div_feight_fee - feight_fee_d),feight_fee_d),
    if(rn=1,benefit_reduce_amount - (sum_div_benefit_reduce_amount -benefit_reduce_amount_d), benefit_reduce_amount_d)
from
(
    select
        od.id,
        od.order_id,
        od.user_id,
        od.sku_id,
        od.sku_name,
        od.order_price,
        od.sku_num,
        od.create_time,
        oi.province_id,
        od.source_type,
        od.source_id,
        round(od.order_price*od.sku_num,2) original_amount_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.final_total_amount,2) final_amount_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.feight_fee,2) feight_fee_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.benefit_reduce_amount,2) benefit_reduce_amount_d,
        row_number() over(partition by od.order_id order by od.id desc) rn,
        oi.final_total_amount,
        oi.feight_fee,
        oi.benefit_reduce_amount,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.final_total_amount,2)) over(partition by od.order_id) sum_div_final_amount,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.feight_fee,2)) over(partition by od.order_id) sum_div_feight_fee,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.benefit_reduce_amount,
2)) over(partition by od.order_id) sum_div_benefit_reduce_amount
    from 
    (
        select * from ods_order_detail where dt='2022-05-10'
    ) od
    join 
    (
        select * from ods_order_info where dt='2022-05-10'
    ) oi
    on od.order_id=oi.id
)t1;        
```

### 查询数据

```sql
select * from dwd_fact_order_detail where dt='2022-05-10' limit 2;
```

## 2.10 加购事实表（周期型快照事实表，每日快照）

由于购物车的数量是会发生变化，所以导增量不合适。

每天做一次快照，导入的数据是全量，区别于事务型事实表是每天导入新增。

周期型快照事实表劣势：存储的数据量会比较大。

解决方案：周期型快照事实表存储的数据比较讲究时效性，时间太久了的意义不大，可以删除以前的数据。

### 建表

```sql
create external table dwd_fact_cart_info(
    `id` string COMMENT '编号',
    `user_id` string  COMMENT '用户id',
    `sku_id` string  COMMENT 'skuid',
    `cart_price` string  COMMENT '放入购物车时价格',
    `sku_num` string  COMMENT '数量',
    `sku_name` string  COMMENT 'sku名称 (冗余)',
    `create_time` string  COMMENT '创建时间',
    `operate_time` string COMMENT '修改时间',
    `is_ordered` string COMMENT '是否已经下单。1为已下单;0为未下单',
    `order_time` string  COMMENT '下单时间',
    `source_type` string COMMENT '来源类型',
    `srouce_id` string COMMENT '来源编号'
) COMMENT '加购事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_cart_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据导入

```sql
insert overwrite table dwd_fact_cart_info partition(dt='2022-05-10')
select
    id,
    user_id,
    sku_id,
    cart_price,
    sku_num,
    sku_name,
    create_time,
    operate_time,
    is_ordered,
    order_time,
    source_type,
    source_id
from ods_cart_info
where dt='2022-05-10';
```

### 查询数据

```
select * from dwd_fact_cart_info where dt='2022-05-10' limit 2;
```

## 2.11收藏事实表（周期型快照事实表，每日快照）

收藏的标记，是否取消，会发生变化，做增量不合适。

每天做一次快照，导入的数据是全量，区别于事务型事实表是每天导入新增。

### 建表

```sql
create external table dwd_fact_favor_info(
    `id` string COMMENT '编号',
    `user_id` string  COMMENT '用户id',
    `sku_id` string  COMMENT 'skuid',
    `spu_id` string  COMMENT 'spuid',
    `is_cancel` string  COMMENT '是否取消',sql
    `create_time` string  COMMENT '收藏时间',
    `cancel_time` string  COMMENT '取消时间'
) COMMENT '收藏事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_favor_info/'
tblproperties ("parquet.compression"="lzo");
```

### 数据数据

```sql
insert overwrite table dwd_fact_favor_info partition(dt='2022-05-10')
select
    id,
    user_id,
    sku_id,
    spu_id,
    is_cancel,
    create_time,
    cancel_time
from ods_favor_info
where dt='2022-05-10';
```

### 查询数据

```sql
select * from dwd_fact_favor_info where dt='2022-05-10' limit 2;
```

## 2.12优惠券领用事实表（累积型快照事实表）

优惠卷的生命周期：领取优惠卷-》用优惠卷下单-》优惠卷参与支付

累积型快照事实表使用：统计优惠卷领取次数、优惠卷下单次数、优惠卷参与支付次数

### 建表

```sql
create external table dwd_fact_coupon_use(
    `id` string COMMENT '编号',
    `coupon_id` string  COMMENT '优惠券ID',
    `user_id` string  COMMENT 'userid',
    `order_id` string  COMMENT '订单id',
    `coupon_status` string  COMMENT '优惠券状态',
    `get_time` string  COMMENT '领取时间',
    `using_time` string  COMMENT '使用时间(下单)',
    `used_time` string  COMMENT '使用时间(支付)'
) COMMENT '优惠券领用事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_coupon_use/'
tblproperties ("parquet.compression"="lzo");
```

###注意：dt是按照优惠卷领用时间get_time做为分区。

### 数据导入

```sql
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
 
insert overwrite table dwd_fact_coupon_use partition(dt)
select
    if(new.id is null,old.id,new.id),
    if(new.coupon_id is null,old.coupon_id,new.coupon_id),
    if(new.user_id is null,old.user_id,new.user_id),
    if(new.order_id is null,old.order_id,new.order_id),
    if(new.coupon_status is null,old.coupon_status,new.coupon_status),
    if(new.get_time is null,old.get_time,new.get_time),
    if(new.using_time is null,old.using_time,new.using_time),
    if(new.used_time is null,old.used_time,new.used_time),
    date_format(if(new.get_time is null,old.get_time,new.get_time),'yyyy-MM-dd')
from
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from dwd_fact_coupon_use
    where dt in
    (
        select
            date_format(get_time,'yyyy-MM-dd')
        from ods_coupon_use
        where dt='2022-05-10'
    )
)old
full outer join
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from ods_coupon_use
    where dt='2022-05-10'
)new
on old.id=new.id;
```

### 查询数据

```sql
select * from dwd_fact_coupon_use where dt='2022-05-10' limit 2;
```

## 2.13系统函数（concat、concat_ws、collect_set、STR_TO_MAP）

1.concat函数

concat函数在连接字符串的时候，只要其中一个是NULL，那么将返回NULL

hive> select concat('a','b');

hive> select concat('a','b',null);

![img](https://img-blog.csdnimg.cn/85d7928eb8104d19b6188ef1e9b3e793.png)



2.concat_ws函数

concat_ws函数在连接字符串的时候，只要有一个字符串不是NULL，就不会返回NULL。concat_ws函数需要指定分隔符。

```sql
hive> select concat_ws('-','a','b');
hive> select concat_ws('-','a','b',null);
hive> select concat_ws('','a','b',null);
```



![img](https://img-blog.csdnimg.cn/b0f7dedc25a44be79f04a9bbfecb1366.png)

 3.collect_set函数

1）建表

```lua
create table stud (name string, area string, course string, score int);
```

2）数据插入

```sql
insert into table stud values('zhang3','bj','math',88);
insert into table stud values('li4','bj','math',99);
insert into table stud values('wang5','sh','chinese',92);
insert into table stud values('zhao6','sh','chinese',54);
insert into table stud values('tian7','bj','chinese',91);
```

3）查询数据

```sql
select * from stud;
```

4）把同一分组的不同行的数据聚合成一个集合

```scss
select course, collect_set(area), avg(score) from stud group by course;
```

![img](https://img-blog.csdnimg.cn/e5ea832a6a214453af16577a1e29d0a4.png)

 5）用下标可以取某一个

```scss
select course, collect_set(area)[0], avg(score) from stud group by course;
```

![img](https://img-blog.csdnimg.cn/973cd297bfa74209a76ab028143867cc.png)

 4.STR_TO_MAP函数

1）STR_TO_MAP(VARCHAR **text**, VARCHAR **listDelimiter**, VARCHAR **keyValueDelimiter**)

2）功能描述

使用listDelimiter将text分隔成K-V对，然后使用keyValueDelimiter分隔每个K-V对，组装成MAP返回。默认listDelimiter为（ ，），keyValueDelimiter为（=）。

3）案例

str_to_map('1001=2022-05-14,1002=2022-05-14',  ','  ,  '=')
输出
{"1001":"2022-05-14","1002":"2022-05-14"}

## 2.14订单事实表（累积型快照事实表）

订单生命周期：创建时间=》支付时间=》取消时间=》完成时间=》退款时间=》退款完成时间。

由于ODS层订单表只有创建时间和操作时间两个状态，不能表达所有时间含义，所以需要关联订单状态表。订单事实表里面增加了活动id，所以需要关联活动订单表。

1.建表

```typescript
create external table dwd_fact_order_info (
    `id` string COMMENT '订单编号',
    `order_status` string COMMENT '订单状态',
    `user_id` string COMMENT '用户id',
    `out_trade_no` string COMMENT '支付流水号',
    `create_time` string COMMENT '创建时间(未支付状态)',
    `payment_time` string COMMENT '支付时间(已支付状态)',
    `cancel_time` string COMMENT '取消时间(已取消状态)',
    `finish_time` string COMMENT '完成时间(已完成状态)',
    `refund_time` string COMMENT '退款时间(退款中状态)',
    `refund_finish_time` string COMMENT '退款完成时间(退款完成状态)',
    `province_id` string COMMENT '省份ID',
    `activity_id` string COMMENT '活动ID',
    `original_total_amount` decimal(16,2) COMMENT '原价金额',
    `benefit_reduce_amount` decimal(16,2) COMMENT '优惠金额',
    `feight_fee` decimal(16,2) COMMENT '运费',
    `final_total_amount` decimal(16,2) COMMENT '订单金额'
) COMMENT '订单事实表'
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_fact_order_info/'
tblproperties ("parquet.compression"="lzo");
```

2.常用函数

```sql
select order_id, concat(order_status,'=', operate_time) from ods_order_status_log where dt='2022-05-10'; 
```

![img](https://img-blog.csdnimg.cn/0e80057c29b54f58a248473c8e12d105.png)

```csharp
select order_id, collect_set(concat(order_status,'=',operate_time)) from ods_order_status_log where dt='2022-05-10' group by order_id;
```

![img](https://img-blog.csdnimg.cn/cf78561546484524802272c32167d91c.png)

```csharp
select order_id, concat_ws(',', collect_set(concat(order_status,'=',operate_time))) from ods_order_status_log where dt='2022-05-10' group by order_id;
```

![img](https://img-blog.csdnimg.cn/0b946b9fb7e24b0396ccdcaecff0d472.png)

```csharp
select order_id, str_to_map(concat_ws(',',collect_set(concat(order_status,'=',operate_time))), ','  ,  '=') from ods_order_status_log where dt='2022-05-10' group by order_id;
```

![img](https://img-blog.csdnimg.cn/c009c0353d324a9aa3cb19257ffae31e.png)

3.数据导入

```sql
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
 
 
 
insert overwrite table dwd_fact_order_info partition(dt)
select
    if(new.id is null,old.id,new.id),
    if(new.order_status is null,old.order_status,new.order_status),
    if(new.user_id is null,old.user_id,new.user_id),
    if(new.out_trade_no is null,old.out_trade_no,new.out_trade_no),
    if(new.tms['1001'] is null,old.create_time,new.tms['1001']),--1001对应未支付状态
    if(new.tms['1002'] is null,old.payment_time,new.tms['1002']),
    if(new.tms['1003'] is null,old.cancel_time,new.tms['1003']),
    if(new.tms['1004'] is null,old.finish_time,new.tms['1004']),
    if(new.tms['1005'] is null,old.refund_time,new.tms['1005']),
    if(new.tms['1006'] is null,old.refund_finish_time,new.tms['1006']),
    if(new.province_id is null,old.province_id,new.province_id),
    if(new.activity_id is null,old.activity_id,new.activity_id),
    if(new.original_total_amount is null,old.original_total_amount,new.original_total_amount),
    if(new.benefit_reduce_amount is null,old.benefit_reduce_amount,new.benefit_reduce_amount),
    if(new.feight_fee is null,old.feight_fee,new.feight_fee),
    if(new.final_total_amount is null,old.final_total_amount,new.final_total_amount),
    date_format(if(new.tms['1001'] is null,old.create_time,new.tms['1001']),'yyyy-MM-dd')
from
(
    select
        id,
        order_status,
        user_id,
        out_trade_no,
        create_time,
        payment_time,
        cancel_time,
        finish_time,
        refund_time,
        refund_finish_time,
        province_id,
        activity_id,
        original_total_amount,
        benefit_reduce_amount,
        feight_fee,
        final_total_amount
    from dwd_fact_order_info
    where dt
    in
    (
        select
          date_format(create_time,'yyyy-MM-dd')
        from ods_order_info
        where dt='2022-05-10'
    )
)old
full outer join
(
    select
        info.id,
        info.order_status,
        info.user_id,
        info.out_trade_no,
        info.province_id,
        act.activity_id,
        log.tms,
        info.original_total_amount,
        info.benefit_reduce_amount,
        info.feight_fee,
        info.final_total_amount
    from
    (
        select
            order_id,
            str_to_map(concat_ws(',',collect_set(concat(order_status,'=',operate_time))),',','=') tms
        from ods_order_status_log
        where dt='2022-05-10'
        group by order_id
    )log
    join
    (
        select * from ods_order_info where dt='2022-05-10'
    )info
    on log.order_id=info.id
    left join
    (
        select * from ods_activity_order where dt='2022-05-10'
    )act
    on log.order_id=act.order_id
)new
on old.id=new.id;
```

4.查询数据

```sql
select * from dwd_fact_order_info where dt='2022-05-10' limit 2;
```

## 2.15用户维度表（拉链表）

用户表中的数据每日既有可能新增，也有可能修改，但修改频率并不高，属于缓慢变化维度，此处采用拉链表存储用户维度数据。

如何使用拉链表

![img](https://img-blog.csdnimg.cn/c935d92331144352a69e1ee3fe93020f.png)

 拉链表形成过程

 ![img](https://img-blog.csdnimg.cn/826649ddc2ba49a593de4a3d85e4c936.png)

![img](https://img-blog.csdnimg.cn/85ff22d6f5a24b1e973f328127ccdf11.png)



拉链表制作过程

步骤0：初始化拉链表（首次独立执行）

（1）建立拉链表

建表

```typescript
create external table dwd_dim_user_info_his(
    `id` string COMMENT '用户id',
    `name` string COMMENT '姓名', 
    `birthday` string COMMENT '生日',
    `gender` string COMMENT '性别',
    `email` string COMMENT '邮箱',
    `user_level` string COMMENT '用户等级',
    `create_time` string COMMENT '创建时间',
    `operate_time` string COMMENT '操作时间',
    `start_date`  string COMMENT '有效开始日期',
    `end_date`  string COMMENT '有效结束日期'
) COMMENT '用户拉链表'
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_user_info_his/'
tblproperties ("parquet.compression"="lzo");
```

初始化拉链表

```sql
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
 
 
insert overwrite table dwd_dim_user_info_his
select
    id,
    name,
    birthday,
    gender,
    email,
    user_level,
    create_time,
    operate_time,
    '2020-06-14',
    '9999-99-99'
from ods_user_info oi
where oi.dt='2022-05-10';
```

步骤1：制作当日变动数据（包括新增，修改）每日执行

（1）如何获得每日变动表

a.最好表内有创建时间和变动时间（Lucky!）

b.如果没有，可以利用第三方工具监控比如canal，监控MySQL的实时变化进行记录（麻烦）。

c.逐行对比前后两天的数据，检查md5(concat(全部有可能变化的字段))是否相同(low)

d.要求业务数据库提供变动流水（人品，颜值）

（2）因为ods_user_info本身导入过来就是新增变动明细的表，所以不用处理

a）数据库中新增2022-05-12一天的数据

b）通过Sqoop把2022-05-12日所有数据导入

mysql_to_hdfs.sh all 2022-05-12

c）ods层数据导入

hdfs_to_ods_db.sh all 2022-05-12

步骤2：先合并变动信息，再追加新增信息，插入到临时表中

1）建立临时表

```typescript
create external table dwd_dim_user_info_his_tmp(
    `id` string COMMENT '用户id',
    `name` string COMMENT '姓名', 
    `birthday` string COMMENT '生日',
    `gender` string COMMENT '性别',
    `email` string COMMENT '邮箱',
    `user_level` string COMMENT '用户等级',
    `create_time` string COMMENT '创建时间',
    `operate_time` string COMMENT '操作时间',
    `start_date`  string COMMENT '有效开始日期',
    `end_date`  string COMMENT '有效结束日期'
) COMMENT '订单拉链临时表'
stored as parquet
location '/warehouse/gmall/dwd/dwd_dim_user_info_his_tmp/'
tblproperties ("parquet.compression"="lzo");
```

2）导入脚本

![img](https://img-blog.csdnimg.cn/441bf5bcfded43a4bea74e7774395c62.png)



```sql
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
insert overwrite table dwd_dim_user_info_his_tmp
select * from 
(
    select 
        id,
        name,
        birthday,
        gender,
        email,
        user_level,
        create_time,
        operate_time,
        '2022-05-12' start_date,
        '9999-99-99' end_date
    from ods_user_info where dt='2022-05-12'
 
    union all 
    select 
        uh.id,
        uh.name,
        uh.birthday,
        uh.gender,
        uh.email,
        uh.user_level,
        uh.create_time,
        uh.operate_time,
        uh.start_date,
        if(ui.id is not null  and uh.end_date='9999-99-99', date_add(ui.dt,-1), uh.end_date) end_date
    from dwd_dim_user_info_his uh left join 
    (
        select
            *
        from ods_user_info
        where dt='2022-05-12'
    ) ui on uh.id=ui.id
)his 
order by his.id, start_date;
```

![img](https://img-blog.csdnimg.cn/0bf0877d0e8c46fc97e4a74179a5fc3e.png)

步骤3：把临时表覆盖给拉链表

1）导入数据

```sql
insert overwrite table dwd_dim_user_info_his 
select * from dwd_dim_user_info_his_tmp;
```

 2）查询导入数据

```sql
select id, start_date, end_date from dwd_dim_user_info_his limit 2;
```

![img](https://img-blog.csdnimg.cn/5f97ba8c9eb340ea8887ceac22de8e6a.png)



## 2.16DWD层业务数据导入脚本

1.编写脚本

vim ods_to_dwd_db.sh

在脚本中填写如下内容

```bash
#!/bin/bash
 
APP=default
hive=/training/hive/bin/hive
 
# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$2" ] ;then
    do_date=$2
else 
    do_date=`date -d "-1 day" +%F`
fi
 
sql1="
set mapreduce.job.queuename=default;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
insert overwrite table ${APP}.dwd_dim_sku_info partition(dt='$do_date')
select  
    sku.id,
    sku.spu_id,
    sku.price,
    sku.sku_name,
    sku.sku_desc,
    sku.weight,
    sku.tm_id,
    ob.tm_name,
    sku.category3_id,
    c2.id category2_id,
    c1.id category1_id,
    c3.name category3_name,
    c2.name category2_name,
    c1.name category1_name,
    spu.spu_name,
    sku.create_time
from
(
    select * from ${APP}.ods_sku_info where dt='$do_date'
)sku
join
(
    select * from ${APP}.ods_base_trademark where dt='$do_date'
)ob on sku.tm_id=ob.tm_id
join
(
    select * from ${APP}.ods_spu_info where dt='$do_date'
)spu on spu.id = sku.spu_id
join 
(
    select * from ${APP}.ods_base_category3 where dt='$do_date'
)c3 on sku.category3_id=c3.id
join 
(
    select * from ${APP}.ods_base_category2 where dt='$do_date'
)c2 on c3.category2_id=c2.id 
join 
(
    select * from ${APP}.ods_base_category1 where dt='$do_date'
)c1 on c2.category1_id=c1.id;
insert overwrite table ${APP}.dwd_dim_coupon_info partition(dt='$do_date')
select
    id,
    coupon_name,
    coupon_type,
    condition_amount,
    condition_num,
    activity_id,
    benefit_amount,
    benefit_discount,
    create_time,
    range_type,
    spu_id,
    tm_id,
    category3_id,
    limit_num,
    operate_time,
    expire_time
from ${APP}.ods_coupon_info
where dt='$do_date';
insert overwrite table ${APP}.dwd_dim_activity_info partition(dt='$do_date')
select
    id,
    activity_name,
    activity_type,
    start_time,
    end_time,
    create_time
from ${APP}.ods_activity_info 
where dt='$do_date';
insert overwrite table ${APP}.dwd_fact_order_detail partition(dt='$do_date')
select
    id,
    order_id,
    user_id,
    sku_id,
    sku_num,
    order_price,
    sku_num,
    create_time,
    province_id,
    source_type,
    source_id,
    original_amount_d,
    if(rn=1,final_total_amount-(sum_div_final_amount-final_amount_d),final_amount_d),
    if(rn=1,feight_fee-(sum_div_feight_fee-feight_fee_d),feight_fee_d),
    if(rn=1,benefit_reduce_amount-(sum_div_benefit_reduce_amount-benefit_reduce_amount_d),benefit_reduce_amount_d)
from
(
    select
        od.id,
        od.order_id,
        od.user_id,
        od.sku_id,
        od.sku_name,
        od.order_price,
        od.sku_num,
        od.create_time,
        oi.province_id,
        od.source_type,
        od.source_id,
        round(od.order_price*od.sku_num,2) original_amount_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.final_total_amount,2) final_amount_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.feight_fee,2) feight_fee_d,
        round(od.order_price*od.sku_num/oi.original_total_amount*oi.benefit_reduce_amount,2) benefit_reduce_amount_d,
        row_number() over(partition by od.order_id order by od.id desc) rn,
        oi.final_total_amount,
        oi.feight_fee,
        oi.benefit_reduce_amount,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.final_total_amount,2)) over(partition by od.order_id) sum_div_final_amount,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.feight_fee,2)) over(partition by od.order_id) sum_div_feight_fee,
        sum(round(od.order_price*od.sku_num/oi.original_total_amount*oi.benefit_reduce_amount,2)) over(partition by od.order_id) sum_div_benefit_reduce_amount
    from 
    (
        select * from ${APP}.ods_order_detail where dt='$do_date'
    ) od
    join 
    (
        select * from ${APP}.ods_order_info where dt='$do_date'
    ) oi
    on od.order_id=oi.id
)t1;
insert overwrite table ${APP}.dwd_fact_payment_info partition(dt='$do_date')
select
    pi.id,
    pi.out_trade_no,
    pi.order_id,
    pi.user_id,
    pi.alipay_trade_no,
    pi.total_amount,
    pi.subject,
    pi.payment_type,
    pi.payment_time,          
    oi.province_id
from
(
    select * from ${APP}.ods_payment_info where dt='$do_date'
)pi
join
(
    select id, province_id from ${APP}.ods_order_info where dt='$do_date'
)oi
on pi.order_id = oi.id;
insert overwrite table ${APP}.dwd_fact_order_refund_info partition(dt='$do_date')
select
    id,
    user_id,
    order_id,
    sku_id,
    refund_type,
    refund_num,
    refund_amount,
    refund_reason_type,
    create_time
from ${APP}.ods_order_refund_info
where dt='$do_date';
insert overwrite table ${APP}.dwd_fact_comment_info partition(dt='$do_date')
select
    id,
    user_id,
    sku_id,
    spu_id,
    order_id,
    appraise,
    create_time
from ${APP}.ods_comment_info
where dt='$do_date';
insert overwrite table ${APP}.dwd_fact_cart_info partition(dt='$do_date')
select
    id,
    user_id,
    sku_id,
    cart_price,
    sku_num,
    sku_name,
    create_time,
    operate_time,
    is_ordered,
    order_time,
    source_type,
    source_id
from ${APP}.ods_cart_info
where dt='$do_date';
insert overwrite table ${APP}.dwd_fact_favor_info partition(dt='$do_date')
select
    id,
    user_id,
    sku_id,
    spu_id,
    is_cancel,
    create_time,
    cancel_time
from ${APP}.ods_favor_info
where dt='$do_date';
insert overwrite table ${APP}.dwd_fact_coupon_use partition(dt)
select
    if(new.id is null,old.id,new.id),
    if(new.coupon_id is null,old.coupon_id,new.coupon_id),
    if(new.user_id is null,old.user_id,new.user_id),
    if(new.order_id is null,old.order_id,new.order_id),
    if(new.coupon_status is null,old.coupon_status,new.coupon_status),
    if(new.get_time is null,old.get_time,new.get_time),
    if(new.using_time is null,old.using_time,new.using_time),
    if(new.used_time is null,old.used_time,new.used_time),
    date_format(if(new.get_time is null,old.get_time,new.get_time),'yyyy-MM-dd')
from
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from ${APP}.dwd_fact_coupon_use
    where dt in
    (
        select
            date_format(get_time,'yyyy-MM-dd')
        from ${APP}.ods_coupon_use
        where dt='$do_date'
    )
)old
full outer join
(
    select
        id,
        coupon_id,
        user_id,
        order_id,
        coupon_status,
        get_time,
        using_time,
        used_time
    from ${APP}.ods_coupon_use
    where dt='$do_date'
)new
on old.id=new.id;
insert overwrite table ${APP}.dwd_fact_order_info partition(dt)
select
    if(new.id is null,old.id,new.id),
    if(new.order_status is null,old.order_status,new.order_status),
    if(new.user_id is null,old.user_id,new.user_id),
    if(new.out_trade_no is null,old.out_trade_no,new.out_trade_no),
    if(new.tms['1001'] is null,old.create_time,new.tms['1001']),--1001对应未支付状态
    if(new.tms['1002'] is null,old.payment_time,new.tms['1002']),
    if(new.tms['1003'] is null,old.cancel_time,new.tms['1003']),
    if(new.tms['1004'] is null,old.finish_time,new.tms['1004']),
    if(new.tms['1005'] is null,old.refund_time,new.tms['1005']),
    if(new.tms['1006'] is null,old.refund_finish_time,new.tms['1006']),
    if(new.province_id is null,old.province_id,new.province_id),
    if(new.activity_id is null,old.activity_id,new.activity_id),
    if(new.original_total_amount is null,old.original_total_amount,new.original_total_amount),
    if(new.benefit_reduce_amount is null,old.benefit_reduce_amount,new.benefit_reduce_amount),
    if(new.feight_fee is null,old.feight_fee,new.feight_fee),
    if(new.final_total_amount is null,old.final_total_amount,new.final_total_amount),
    date_format(if(new.tms['1001'] is null,old.create_time,new.tms['1001']),'yyyy-MM-dd')
from
(
    select
        id,
        order_status,
        user_id,
        out_trade_no,
        create_time,
        payment_time,
        cancel_time,
        finish_time,
        refund_time,
        refund_finish_time,
        province_id,
        activity_id,
        original_total_amount,
        benefit_reduce_amount,
        feight_fee,
        final_total_amount
    from ${APP}.dwd_fact_order_info
    where dt
    in
    (
        select
          date_format(create_time,'yyyy-MM-dd')
        from ${APP}.ods_order_info
        where dt='$do_date'
    )
)old
full outer join
(
    select
        info.id,
        info.order_status,
        info.user_id,
        info.out_trade_no,
        info.province_id,
        act.activity_id,
        log.tms,
        info.original_total_amount,
        info.benefit_reduce_amount,
        info.feight_fee,
        info.final_total_amount
    from
    (
        select
            order_id,
            str_to_map(concat_ws(',',collect_set(concat(order_status,'=',operate_time))),',','=') tms
        from ${APP}.ods_order_status_log
        where dt='$do_date'
        group by order_id
    )log
    join
    (
        select * from ${APP}.ods_order_info where dt='$do_date'
    )info
    on log.order_id=info.id
    left join
    (
        select * from ${APP}.ods_activity_order where dt='$do_date'
    )act
    on log.order_id=act.order_id
)new
on old.id=new.id;
"
 
sql2="
insert overwrite table ${APP}.dwd_dim_base_province
select 
    bp.id,
    bp.name,
    bp.area_code,
    bp.iso_code,
    bp.region_id,
    br.region_name
from ${APP}.ods_base_province bp
join ${APP}.ods_base_region br
on bp.region_id=br.id;
"
 
sql3="
insert overwrite table ${APP}.dwd_dim_user_info_his_tmp
select * from 
(
    select 
        id,
        name,
        birthday,
        gender,
        email,
        user_level,
        create_time,
        operate_time,
        '$do_date' start_date,
        '9999-99-99' end_date
    from ${APP}.ods_user_info where dt='$do_date'
    union all 
    select 
        uh.id,
        uh.name,
        uh.birthday,
        uh.gender,
        uh.email,
        uh.user_level,
        uh.create_time,
        uh.operate_time,
        uh.start_date,
        if(ui.id is not null  and uh.end_date='9999-99-99', date_add(ui.dt,-1), uh.end_date) end_date
    from ${APP}.dwd_dim_user_info_his uh left join 
    (
        select
            *
        from ${APP}.ods_user_info
        where dt='$do_date'
    ) ui on uh.id=ui.id
)his 
order by his.id, start_date;
insert overwrite table ${APP}.dwd_dim_user_info_his 
select * from ${APP}.dwd_dim_user_info_his_tmp;
"
 
case $1 in
"first"){
    $hive -e "$sql1$sql2"
};;
"all"){
    $hive -e "$sql1$sql3"
};;
esac
```

增加脚本执行权限：chmod 777 ods_to_dwd_db.sh

2.脚本使用说明

1）初次导入

（1）时间维度表：参照2.5节

（2）用户维度表：参照2.15节拉链表初始化

（3）其余表

初次导入时，脚本的第一个参数应为first，线上环境不传第二个参数，自动获取前一天日期。

```sql
ods_to_dwd_db.sh first 2022-05-10
```

2）每日定时导入

每日定时导入，脚本的第一个参数应为all，线上环境不传第二个参数，自动获取前一天日期。

```scss
ods_to_dwd_db.sh all 2022-05-10
```