- [Hive结合shell脚本企业实战用法 ](https://www.cnblogs.com/saowei/p/15928996.html)

# 1 Hive结合Shell脚本企业实战用法

## 1.1 Linux设置定时任务

```bash
编辑定时任务
[root@master ]#crontab -e

* * * * *

第1个*：表示分钟
第2个*：表示小时
第3个*：表示星期几
第4个*：表示几号
第5个*：表示月份

举例：
每分钟执行一次hql.sh脚本，将其日志存放到1.log目录中
*/1 * * * * /usr/local/soft/scripts/hql.sh >> /usr/local/soft/scripts/logs/1.log
```

## 1.2 Hive结合shell脚本使用

hive -e “SQL语句”

```sql
举例：
[root@master ~]# hive -e “select * from test1.students limit 1”

执行结果：
1500100001	施笑槐	22	女	文科六班
```

hive -f sql文件

```sql
当我们的SQL比较复杂，不方便写在hive -e 这一行，我们可以先把SQL语句写下文件里
举例：
[root@master scripts]# vim stu.sql --------编辑sql文件（在里面写sql语句）
select * from test1.students limit 1;

[root@master scripts]# hive -f stu.sql ----执行sql文件
执行结果：
1500100001	施笑槐	22	女	文科六班
```

Hive 在 Linux 的 Shell 中的使用

第一种：

> ```bash
> hive -e "后面直接加一条SQL"
> ```

```bash
hive -e "select * from test1.students limit 10"
```

第二种：

```powershell
hive -f hql文件路径
```

> 将HQL写在一个文件里，再使用 -f 参数指定该文件

## 1.3 编辑 `.sh` 脚本文件

> **在shell脚本中获取Linux系统当前日期时间**
>
> **`time=$(date "+%Y-%m-%d %H:%M:%S")`**
>
> **`sed -i "s/A/B/" 文件名` 用 sed 的修改结果直接修改读取数据的文件，而不是由屏幕输出动作**
>
> > A -- 需要被修改的数据
> >
> > B -- 修改内容

```bash
# 例如

# hql.sh文件  --  hive -e

#!/bin/sh
#date="2022-02-22"
date=$(date "+%Y-%m-%d")
sql1="select * from students_pt where pt='${date}'"
hive -e ${sql1}
echo $sql1

# hql.sh文件  --  hive -f

#!/bin/sh
#date="2022-02-22"
date=$(date "+%Y%m%d")
sed -i "s/!everydate!/${date}/" /usr/local/soft/scripts/stu_pt.sql
cat /usr/local/soft/scripts/stu_pt.sql
hive -f /usr/local/soft/scripts/stu_pt.sql
sed -i "s/${date}/!everydate!/" /usr/local/soft/scripts/stu_pt.sql
cat /usr/local/soft/scripts/stu_pt.sql


// stu_pt.sql  --  SQL文件
select	* from test1.students_pt1 where pt='!everydate!';
```

## 1.4 添加定时任务

> 具体内容参考shell编程学习笔记中的`定时器`
>
> **注意 `xxx.sh` 需要有可执行权限**
>
> 添加执行权限 `chmod a+x 文件名`

**`通过crontab -e添加定时任务`**

**:wq 即 添加 并 开启定时任务**

```javascript
// 将输出结果重定向
*(分) *(时) *(星期) *(几号) *(月份) /xxx.sh >> /xxx

// 例如 保存 :wq 即会开启定时任务
*/1 * * * * /usr/local/soft/scripts/hql.sh >> /usr/local/soft/scripts/logs/1
```

# 2 Hive定时执行脚本

- [hive 定时执行脚本](https://www.cnblogs.com/cstark/p/15075905.html)

hive中，需要每天定时跑一些sql的脚本或者其他脚本，可以通过调用sh脚本

1.新建一个sh文件

```bash
[root@master log]# vim wh_hive_daily.sh
```

2.添加以下内容 （可以直接复制，然后修改sql里面的内容即可）

```bash
#!/bin/bash
APP=uiopdb
hive=/opt/soft/hive/bin/hive
if [-n $1] ;then
    do_date=$1
else
    do_date=`date -d "-1 day" +%F`
fi

sql="
insert into uiopdb.sx_trade_his_detail partition(dt='$do_date')
select a.*  from uiopdb.sx_trade_his a where   date_format(trade_time,'yyyy-MM-dd')='$do_date';
"

$hive -e "$sql" >>/opt/soft/hive/job/upy/log/log.$do_date
```

其中：

在调用脚本时，如果不指定参数，就可以直接默认昨日时间（用于正常每天运行）

如果需要指定某一天的日期时，在调用脚本时，指定其时间即可

```
还有，>>/opt/soft/hive/job/upy/log/log.$do_date 这个日志可以不写
```

正常运行昨日数据

```
[root@master upy]# wh_hive_daily.sh
```

指定日期

```
[root@master upy]# wh_hive_daily.sh 2021-07-28
```

# 3 ODS层业务表每日数据装载脚本

vim hdfs_to_ods_db.sh

（1）在脚本中填写如下内容

```bash
#!/bin/bash
 
APP=gmall
 
# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
 
if [ -n "$2" ] ;then
    do_date=$2
else 
    do_date=`date -d "-1 day" +%F`
fi
 
ods_user_info="
load data inpath '/origin_data/$APP/db/user_info/$do_date' OVERWRITE into table ${APP}.ods_user_info partition(dt='$do_date');"
 
case $1 in
 
    "ods_user_info"){
        hive -e "$ods_user_info"
    };;
    };;
    "all"){
        hive -e "$ods_user_info"
    };;
esac
```

（2）增加执行权限

```bash
[muzili@hadoop102 bin]$ chmod +x hdfs_to_ods_db.sh
```

（3）执行脚本

```bash
[muzili@hadoop102 bin]$ hdfs_to_ods_db.sh all 2020-09-16
```

（4）查看数据是否导入成功

其他：

```sql
-- ap_raw
 
-- 建表
create external table if not exists ods.ap_raw(
serverreceivetimestamp bigint comment '数据接收时的时间戳',
gbno string comment '国标码',
sn string comment '设备sn',
deveice_mac string comment '设备MAC',
macs array<struct<timestamp:bigint,mac:string,channel:int,signal:int>> comment '采集到的mac信息'
) comment 'AP数据(真实MAC)'
partitioned by(
filedate string comment '文件所属日期'
)stored as parquet;
 
-- 导数据
DATE=$(date -d '-1 day' +"%Y%m%d")
alter table ods.bus_gps add if not exists partition(filedate='$DATE') location '/user/hadoop/GongAnV2/bus_gps/$DATE';
```

# 4 DWD层业务数据每日装载脚本

- [Hive的dwd层建表语句及加载数据](https://blog.csdn.net/qq_56870570/article/details/120328231)

vim ods_to_dwd_db.sh

（1）在脚本中填写如下内容:

```bash
#!/bin/bash
 
APP=gmall
# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$2" ] ;then
    do_date=$2
else 
    do_date=`date -d "-1 day" +%F`
fi
 
 
# 假设某累积型快照事实表，某天所有的业务记录全部完成，则会导致9999-99-99分区的数据未被覆盖，从而导致数据重复，该函数根据9999-99-99分区的数据的末次修改时间判断其是否被覆盖了，如果未被覆盖，就手动清理
clear_data(){
    current_date=`date +%F`
    current_date_timestamp=`date -d "$current_date" +%s`
 
    last_modified_date=`hadoop fs -ls /warehouse/gmall/dwd/$1 | grep '9999-99-99' | awk '{print $6}'`
    last_modified_date_timestamp=`date -d "$last_modified_date" +%s`
 
    if [[ $last_modified_date_timestamp -lt $current_date_timestamp ]]; then
        echo "clear table $1 partition(dt=9999-99-99)"
        hadoop fs -rm -r -f /warehouse/gmall/dwd/$1/dt=9999-99-99/*
    fi
}
 
dwd_order_info="
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
set hive.exec.dynamic.partition.mode=nonstrict;
insert overwrite table ${APP}.dwd_order_info partition(dt)
select
    nvl(new.id,old.id),
    nvl(new.final_amount,old.final_amount),
    case
        when new.cancel_time is not null then date_format(new.cancel_time,'yyyy-MM-dd')
        when new.finish_time is not null and date_add(date_format(new.finish_time,'yyyy-MM-dd'),7)='$do_date' and new.refund_time is null then '$do_date'
        when new.refund_finish_time is not null then date_format(new.refund_finish_time,'yyyy-MM-dd')
        when new.expire_time is not null then date_format(new.expire_time,'yyyy-MM-dd')
        else '9999-99-99'
    end
from
(
    select
        id,
        final_amount
    from ${APP}.dwd_order_info
    where dt='9999-99-99'
)old
full outer join
(
    select
        oi.id,
        final_amount
    from
    (
        select
            *
        from ${APP}.ods_order_info
        where dt='$do_date'
    )oi
    left join
    (
        select
            order_id,
            str_to_map(concat_ws(',',collect_set(concat(order_status,'=',operate_time))),',','=') ts
        from ${APP}.ods_order_status_log
        where dt='$do_date'
        group by order_id
    )times
    on oi.id=times.order_id
)new
on old.id=new.id;"
 
case $1 in
    dwd_order_info )
        hive -e "$dwd_order_info"
        clear_data dwd_order_info
    ;;
    all )
        hive -e "$dwd_order_info$dwd_order_detail$dwd_payment_info$dwd_cart_info$dwd_comment_info$dwd_favor_info$dwd_coupon_use$dwd_order_refund_info$dwd_refund_payment"
        clear_data dwd_order_info
        clear_data dwd_payment_info
        clear_data dwd_coupon_use
        clear_data dwd_refund_payment
    ;;
esac
```

（2）增加脚本执行权限

```bash
[muzili@hadoop102 bin]$ chmod 777 ods_to_dwd_db.sh
```

（3）执行脚本

```bash
[muzili@hadoop102 bin]$ ods_to_dwd_db.sh all 2021-09-16
```

（4）查看数据是否导入成功

```sql
-- 导入数据
insert overwrite table dwd.ap_raw_detail PARTITION (filedate='20210905')
select serverreceivetimestamp,gbno,sn,deveice_mac,
mac_info.timestamp,mac_info.mac,mac_info.channel,mac_info.signal
from ods.ap_raw lateral view explode(macs) tmp_table as mac_info
where filedate='20210905';
```

# 5 hive导入导出数据自动化的shell脚本

- [hive导入导出数据自动化的shell脚本](https://blog.csdn.net/lijian972/article/details/123792167)

## 5.1 从mysql导入到hive的ods层的shell脚本

传参表示要导入的数据分期日期，sqoop导入，实例：

```powershell
#!/bin/bash
export SQOOP_HOME=/usr/bin/sqoop
if [ $# == 1 ]
   then
      dateStr=$1
   else
      dateStr=`date -d '-1 day' +'%Y-%m-%d'`
fi

dateNowStr=`date +'%Y-%m-%d'`

yearStr=`date -d ${dateStr} +'%Y'`
monthStr=`date -d ${dateStr} +'%m'`

jdbcUrl='jdbc:mysql://192.168.88.80:3306/nev'
username='root'
password='123456'
m='1'

${SQOOP_HOME} import \
--connect ${jdbcUrl} \
--username ${username} \
--password ${password} \
--query "SELECT 
id,create_date_time,session_id,sid,create_time,seo_source,seo_keywords,ip,
AREA,country,province,city,origin_channel,USER AS user_match,manual_time,begin_time,end_time,
last_customer_msg_time_stamp,last_agent_msg_time_stamp,reply_msg_count,
msg_count,browser_name,os_info, '${dateNowStr}' AS starts_time  
FROM web_chat_ems_${yearStr}_${monthStr} WHERE create_time BETWEEN '${dateStr} 00:00:00' AND '${dateStr} 
23:59:59' and \$CONDITIONS" \
--hcatalog-database itcast_ods \
--hcatalog-table web_chat_ems \
-m ${m}

${SQOOP_HOME} import \
--connect ${jdbcUrl} \
--username ${username} \
--password ${password} \
--query "SELECT 
        temp2.*, '${dateNowStr}' AS start_time
FROM (SELECT id FROM web_chat_ems_${yearStr}_${monthStr} WHERE create_time BETWEEN '${dateStr} 00:00:00' 
AND '${dateStr} 23:59:59') temp1
        JOIN web_chat_text_ems_${yearStr}_${monthStr} temp2 ON temp1.id = temp2.id where 1=1 and \$CONDIT
IONS" \
--hcatalog-database itcast_ods \
--hcatalog-table web_chat_text_ems \
-m ${m}
```

## 5.2 执行从ods层到dwd层或者从dwd层到dwb层的shell脚本

语法：./hive -e|-f ‘sql语句|SQL脚本’ -S
例子：

```powershell
#!/bin/bash
export HIVE_HOME=/usr/bin/hive
if [ $# == 1 ]
then
  dateStr=$1
  else
     dateStr=`date +'%Y-%m-%d'`
fi

sqlStr="
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.exec.compress.intermediate=true;
set hive.exec.compress.output=true;
set hive.exec.orc.compression.strategy=COMPRESSION;

insert into table itcast_dwd.visit_consult_dwd partition(yearinfo,quarterinfo,monthinfo,dayinfo)
select
    wce.session_id,
    wce.sid,
    unix_timestamp(wce.create_time) as create_time,  
    wce.seo_source,
    wce.ip,
    wce.area,
    wce.msg_count,
    wce.origin_channel,
    wcte.referrer,
    wcte.from_url,
    wcte.landing_page_url,
    wcte.url_title,
    wcte.platform_description,
    wcte.other_params,
    wcte.history,
    substr(wce.create_time,12,2) as hourinfo,
    substr(wce.create_time,1,4) as yearinfo, 
    quarter(wce.create_time) as quarterinfo,
    substr(wce.create_time,6,2) as monthinfo,
    substr(wce.create_time,9,2) as dayinfo
from (select * from itcast_ods.web_chat_ems where starts_time='${dateStr}') wce join (select * from itcast_ods.web_chat_text_ems where start_time='${dateStr}') wcte   
    on wce.id = wcte.id;
"

${HIVE_HOME} -e "${sqlStr}" -S
```

## 5.3 执行从dws层到mysql层的shell脚本

增量数据导出操作说明：
在执行导出的时候, 也需要将mysql中之前的统计的当年当季度和当月的结果数据删除, 然后重新导入操作
此时我们处理的方案, 要进行简化一些, 受影响最大范围当年的数据, 可以直接将当年的统计结果数据全部都删除, 然后重新到DWS层当年的所有数据

```powershell
#!/bin/bash
export SQOOP_HOME=/usr/bin/sqoop
if [ $# == 1 ]
then
   TD_DATE=$1  
else
   TD_DATE=`date -d '-1 day' +'%Y-%m-%d'`
fi

TD_YEAR=`date -d ${TD_DATE} +%Y`

mysql -uroot -p123456 -h192.168.52.150 -P3306 -e "delete from scrm_bi.visit_dws where yearinfo='$TD_YEAR'; delete from scrm_bi.consult_dws where yearinfo='$TD_YEAR';"

jdbcUrl='jdbc:mysql://192.168.88.80:3306/scrm_bi?useUnicode=true&characterEncoding=utf-8'
username='root'
```

# 6 Shell脚本循环跑多天的Hive脚本

```bash
#!/bin/bash

start_date=2021-11-04
end_date=2021-11-06
while [[ $start_date < $end_date ]]
do
	date=`date -d "$start_date" +"%Y-%m-%d"`
	hive -e"
	select * from  table ;"
	echo "$date执行完毕"
	start_date=`date -d " +1 day $date" +"%Y-%m-%d"`
done;
```

