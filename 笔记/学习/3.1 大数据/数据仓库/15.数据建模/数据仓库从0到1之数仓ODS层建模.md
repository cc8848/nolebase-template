- [数据仓库从0到1之数仓ODS层建模 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/386622052)
- [数据仓库从0到1之数仓DWD层 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/387970276)

## ODS层用户日志数据和电商业务数据模拟生成导入

### ODS层用户日志数据生成及导入

在前面的[数仓业务逻辑流程分析](https://link.zhihu.com/?target=https%3A//www.everweekup.com/2021/07/02/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E4%BB%8E0%E5%88%B01%E4%B9%8B%E4%B8%9A%E5%8A%A1%E9%80%BB%E8%BE%91%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%9001/)一文中，我们分析了电商数仓的数据来源有两种，一种是用户日志数据json格式信息，一种是存储在mysql中的业务数据表，因此在这里我们要模拟生成这两种类型的数据。

### 用户日志数据生成

这里我们使用[数仓业务逻辑流程分析](https://link.zhihu.com/?target=https%3A//www.everweekup.com/2021/07/02/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E4%BB%8E0%E5%88%B01%E4%B9%8B%E4%B8%9A%E5%8A%A1%E9%80%BB%E8%BE%91%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%9001/)一文中提到的脚本工具来生成数据。

先进入用户日志数据产生脚本所在的目录：

```bash
[dw@hadoop117 applog]$ pwd
/opt/module/applog
[dw@hadoop117 applog]$ ls
application.properties             log          path.json
gmall2020-mock-log-2020-05-10.jar  logback.xml
[dw@hadoop117 applog]$ cat application.properties 
# 外部配置打开
logging.config=./logback.xml

#业务日期，即产生的json数据的业务日期
mock.date=2020-06-15

#模拟数据发送模式
mock.type=log
#mock.type=http
#http模式下，发送的地址
mock.url=http://localhost:8080/applog

#启动次数
mock.startup.count=100
#设备最大值
mock.max.mid=50
#会员最大值
mock.max.uid=500
#商品最大值
mock.max.sku-id=10
#页面平均访问时间
mock.page.during-time-ms=20000
#错误概率 百分比
mock.error.rate=3
#每条日志发送延迟 ms
mock.log.sleep=10
#商品详情来源  用户查询，商品推广，智能推荐, 促销活动
mock.detail.source-type-rate=40:25:15:20
```

再编写一个脚本用于启动节点上负责产生日志数据的机器的日志生成进程：

```bash
[dw@hadoop116 applog]$ cat /home/dw/bin/log.sh 
for i in hadoop116 hadoop117;do
    echo "==========$i=========="
    ssh $i "cd /opt/module/applog;java -jar gmall2020-mock-log-2020-05-10.jar >/dev/null 2>&1 &"
done
```

我这里拿了两台节点用于日志数据的生成，这并不是硬性要求，可以根据自己物理机配置选择。

这里我们运行脚本，在两台节点上，生成业务时间为2020-06-15的日志数据：

```bash
[dw@hadoop116 applog]$ log.sh 
==========hadoop116==========
==========hadoop117==========
[dw@hadoop116 log]$ pwd
/opt/module/applog/log
[dw@hadoop116 log]$ ls
app.2021-07-04.log
```

可以看到我们的日志数据生成了，要注意日志数据名称中的日期是根据当前机器的时间生成的，而用户日志的数据日期则需要到`application.properties`脚本里去配置。

### 用户日志数据采集

这里我们使用在[日志采集平台搭建](https://link.zhihu.com/?target=https%3A//www.everweekup.com/2021/05/06/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E4%BB%8E0%E5%88%B01%E4%B9%8B%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E5%B9%B3%E5%8F%B0%E6%90%AD%E5%BB%BA02/)一文中编写的日志数据采集脚本，来对生成的日志数据进行采集：

```bash
[dw@hadoop116 log]$ log-cluster.sh start
================== 启动 集群 ==================
---------- zookeeper hadoop116 启动 ------------
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg
Starting zookeeper ... already running as process 2350.
---------- zookeeper hadoop117 启动 ------------
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg
Starting zookeeper ... already running as process 2269.
---------- zookeeper hadoop118 启动 ------------
ZooKeeper JMX enabled by default
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg
Starting zookeeper ... already running as process 1930.
====================
Starting namenodes on [hadoop116]
hadoop116: namenode is running.
Starting datanodes
hadoop116: datanode is running.
hadoop117: datanode is running.
hadoop118: datanode is running.
Starting secondary namenodes [hadoop118]
hadoop118: secondarynamenode is running.
Starting resourcemanager
Starting nodemanagers
resourcemanager is running.
 --------启动 hadoop116 Kafka-------
 --------启动 hadoop117 Kafka-------
 --------启动 hadoop118 Kafka-------
==========hadoop116==========
==========hadoop117==========
==========hadoop118==========
```

之后，我们登陆到HDFS的web界面，查看对应路径下是否有数据产生：

![img](https://pic3.zhimg.com/80/v2-2d83c3d3c97a41fe76fe89e3ce11529e_1440w.webp)

至此，用户日志数据采集完成。

## ODS层mysql业务数据生成及导入

### ODS层业务数据生成及导入

### 业务数据生成

这里我们使用[数仓业务逻辑流程分析](https://link.zhihu.com/?target=https%3A//www.everweekup.com/2021/07/02/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E4%BB%8E0%E5%88%B01%E4%B9%8B%E4%B8%9A%E5%8A%A1%E9%80%BB%E8%BE%91%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%9001/)一文中提到的脚本工具来生成mysql业务数据。

进入到脚本路径下，修改popertities文件里的业务数据生成时间：

```bash
[dw@hadoop116 db_log]$ ls
application.properties  gmall2020-mock-db-2020-05-18.jar
[dw@hadoop116 db_log]$ pwd
/opt/module/db_log
[dw@hadoop116 db_log]$ cat application.properties 
```

使得`mock.date=2020-06-15`。

这里写一个脚本用于生成业务数据到mysql，并执行：

```bash
[dw@hadoop116 db_log]$ 
[dw@hadoop116 db_log]$ cat /home/dw/bin/data_2_mysql.sh 
echo "==========开始生成业务数据到mysql=========="
ssh hadoop116 "cd /opt/module/db_log;java -jar gmall2020-mock-db-2020-05-18.jar >/dev/null 2>&1 &"
echo "==========业务数据生成完毕=========="

[dw@hadoop116 db_log]$ data_2_mysql.sh 
==========开始生成业务数据到mysql==========
==========业务数据生成完毕==========
```

连接mysql所在的节点，查看数据是否生成：



![img](https://pic3.zhimg.com/80/v2-e47827f4b6e40699e22d1a5d39b88a1a_1440w.webp)





### 业务数据导入HDFS

将数据从mysql导入到hdfs上，需要用到数据迁移的工具，这里我们使用Sqoop。



### Sqoop安装

不借助于工具实现从MySQL读数据（JDBC），HDFS的IO操作，我们也可以通过自己写java程序实现数据导入MySQL，但这样过于麻烦，开发成本较高，于是，sqoop、datax等工具被实现出来。

Sqoop能够实现关系型数据库和Hadoop生态之间数据的双向传输。即MySQLóHDFS数据双向传输。

将MySQL数据传输到HDFS时，Sqoop会提交一个任务，调用MapReduce，不过只执行Map，主要是因为Sqoop是为了传输数据，不是为了分析，只需要一读一写，不需要计算分析，也就不需要reduce。其次是因为shuffle消耗时间和性能。Sqoop自定义了inputformat（能从mysql、oracle读数据的）和outputformat。

Sqoop有两个版本，一个1.x，一个2.x，两者版本不兼容，相当于两个并行的分支版本。



![img](https://pic2.zhimg.com/80/v2-b6e3b3ba0c38815a394c520189c12ad9_1440w.webp)



Sqoop2.x目前的功能还不齐全，还不用于生产环境的部署。

### 下载并解压

1.下载地址：[http://mirrors.hust.edu.cn/apache/sqoop/1.4.6/](https://link.zhihu.com/?target=http%3A//mirrors.hust.edu.cn/apache/sqoop/1.4.6/)

2.上传安装包sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz到hadoop102的/opt/software路径中

3.解压sqoop安装包到指定目录，如：

```bash
[atguigu@hadoop102 software]$ tar -zxf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz -C /opt/module/
```

4.解压sqoop安装包到指定目录，如：

```bash
[atguigu@hadoop102 module]$ mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha/ sqoop
```

### 修改配置文件

1.进入到/opt/module/sqoop/conf目录，重命名配置文件

```bash
[atguigu@hadoop102 conf]$ mv sqoop-env-template.sh sqoop-env.sh
```

2.修改配置文件

```bash
[atguigu@hadoop102 conf]$ vim sqoop-env.sh 
```

增加如下内容：

```text
# # 这两个变量在hadoop1.x是分开的，2.x之后放在了一个目录
# sqoop是需要MR的，所以需要Hadoop的环境
export HADOOP_COMMON_HOME=/opt/module/hadoop-3.1.3
export HADOOP_MAPRED_HOME=/opt/module/hadoop-3.1.3
# Sqoop为了把关系型数据库传输到HDFS上，这里首先sqoop将数据写到一个路径，然后hive通过load把数据加载到hdfs，实现hdfs和关系型数据库间数据互通
export HIVE_HOME=/opt/module/hive
# 因为数据也可以写入hbase，所以需要zk的配置，找到hbase的regionserver
export ZOOKEEPER_HOME=/opt/module/zookeeper-3.5.7
export ZOOCFGDIR=/opt/module/zookeeper-3.5.7/conf
```

### 拷贝JDBC驱动

1.将mysql-connector-java-5.1.48.jar 上传到/opt/software路径

2.进入到/opt/software/路径，拷贝jdbc驱动到sqoop的lib目录下。

```bash
[atguigu@hadoop102 software]$ cp mysql-connector-java-5.1.48.jar /opt/module/sqoop/lib/
```

### 验证Sqoop

我们可以通过某一个command来验证sqoop配置是否正确：

```bash
[atguigu@hadoop102 sqoop]$ bin/sqoop help
```

出现一些Warning警告（警告信息已省略），并伴随着帮助命令的输出：

```text
Available commands:
  codegen            Generate code to interact with database records
  create-hive-table     Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
# sqoop以hadoop为中心
  import             Import a table from a database to HDFS
  import-all-tables     Import tables from a database to HDFS
  import-mainframe    Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases        List available databases on a server
  list-tables           List available tables in a database
  merge              Merge results of incremental imports
  metastore           Run a standalone Sqoop metastore
  version            Display version information
```

### 测试Sqoop是否能够成功连接数据库

```bash
[atguigu@hadoop102 sqoop]$ bin/sqoop list-databases --connect jdbc:mysql://hadoop102:3306/ --username root --password 000000
# 这里不需要—-driver指定驱动了，因为现在会自动识别使用哪个驱动
```

出现如下输出：

```text
information_schema
metastore
mysql
oozie
performance_schema
```

### Sqoop数据导入脚本

```text
说明1：

[ -n 变量值 ] 判断变量的值，是否为空

-- 变量的值，非空，返回true

-- 变量的值，为空，返回false
说明2：

查看date命令的使用，[atguigu@hadoop102 ~]$ date --help

sqoop导入数据脚本：
```

> 补充：
> Hive中的Null在底层是以“\N”来存储，而MySQL中的Null在底层就是Null，为了保证数据两端的一致性。
> 在导出数据时采用--input-null-string和--input-null-non-string两个参数。
> 导入数据时采用--null-string和--null-non-string。

```text
# sqoop导入mysql数据脚本（如果要执行得去掉注释）
# 换行符在shell脚本里在执行时会被解析掉后将下一行的命令连接起来形成一个完整的命令。所以：
# # 换行符前面要有空格，要不然连成一串会识别不了
# # 换行符后面不能有空格
bin/sqoop import \
--connect jdbc:mysql://hadoop116:3306/gmall \
--username root \
--password 888888 \
# 指明要导入的表,这样写导入的是全表，可以指定导入列
--table user_info \
# shell里要加入引号，得看命令有无空格，且想将命令看成整体才需加引号
--columns id,login_name \
# 筛选数据
# 加引号是单引号还是双引号，看有没有引入外部变量，如果有外部引入则要加双引，双引号才会解析字符串里变量
--where "id >= 10 and id <= 30" \
# 因为前面配置了hadoop_home所以不需要hdfs://
--target-dir /test \
# 如果输出路径存在，任务直接删除目录，避免报错
--delete-target-dir \ 
# 优化相关参数
# # 导数据时map的个数，之前map个数是分片策略决定的，分片策略是由inputformat决定的，但是sqoop的inputformat是自己定义的，根据参数配置去分片。默认4个
# 几个map结果就有多少个文件
--num-mappers 2 \
# 分隔符
--fields-terminated-by '\t' \
# 按照谁切片
# sqoop分片策略，分片策略首先看你配置分片个数的参数，假设是2，
# 那么它会按照参数2，根据你分片依赖的字段进行分片，首先求max id和min id然后取得中间值，
# 然后会将最小id到中间值分一片由一个map读取，中间值到最大值分一片由另一个map读取。
--split-by id \
# sqoop可直接写sql语句 称为free-query
./bin/sqoop import \
--connect jdbc:mysql://hadoop116:3306/gmall \
--username root \
--password 888888 \
# where子句中必须有$CONDITIONS，$CONDITIONS作为一个占位符，由于由切片，分别在切片中执行sql语句，当第一个分片执行该sql
# 语句，则将分片条件带入sql语句替换$CONDITIONS。
# 这里在$CONDITIONS前要加”\”避免shell将其解析。(双引里的变量会被解析除非转义或用单引号)
--query "select * from user_info where id>=10 and id <= 30 and \$CONDITIONS"
# --table user_info \
# --columns id,login_name \
# --where "id >= 10 and id <= 30" \
--target-dir /test \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by '\t' \
--split-by id \
```

下面编写业务数据导入HDFS的sqoop脚本，脚本的一些内容涉及到数据同步策略，将会在下面展开描述，这里先放脚本。

```text
#!/bin/bash
# shell是逐行运行的，想调用函数得把函数写到前面
# ``:将命令执行结果返回给变量，而""或者''都不行，返回的只是命令字符
$do_date=`date -d '-1 day' +%F`
import_data(){
    ./bin/sqoop import \
    --connect jdbc:mysql://hadoop116:3306/gmall \
    --username root \
    --password 888888 \
    # 这里的"$1"是函数的参数不是脚本的参数
    --query "$2 and \$CONDITIONS"
    --target-dir /origin_data/gmall/db/$1/$do_date \
    --delete-target-dir \
    --num-mappers 1 \
    --fields-terminated-by '\t'
    # 配置压缩格式
    --compress \
    --compression-codec lzop \
    # hive能够识别的空值格式是"\N"，所以需要把mysql的null转换成"\N"，这里的两个\起到了转义，因为mr底层是java。
    --null-string '\\N' \
    --null-non-string '\\N'
}

# 多少张表写多少行导入数据的语句
import_data order_info "select id, price, amount from order_info where "

#! /bin/bash

APP=gmall
sqoop=/opt/module/sqoop/bin/sqoop
# -n:-nonull如果$2参数不是空，则使用该参数。应用场景：假如某一天的脚本失败了，想要导某一天的数据，就需要自行传入参数
if [ -n "$2" ] ;then
    do_date=$2
else
    do_date=`date -d '-1 day' +%F`
fi

import_data(){
$sqoop import \
--connect jdbc:mysql://hadoop102:3306/$APP \
--username root \
--password 000000 \
--target-dir /origin_data/$APP/db/$1/$do_date \
--delete-target-dir \
--query "$2 and  \$CONDITIONS" \
--num-mappers 1 \
--fields-terminated-by '\t' \
--compress \
--compression-codec lzop \
--null-string '\\N' \
--null-non-string '\\N'
# \用于转义\（java代码识别）

hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer /origin_data/$APP/db/$1/$do_date
}

import_order_info(){
  import_data order_info "select
                            id, 
                            final_total_amount, 
                            order_status, 
                            user_id, 
                            out_trade_no, 
                            create_time, 
                            operate_time,
                            province_id,
                            benefit_reduce_amount,
                            original_total_amount,
                            feight_fee      
                        from order_info
                        where (date_format(create_time,'%Y-%m-%d')='$do_date' 
                        or date_format(operate_time,'%Y-%m-%d')='$do_date')"
}

import_coupon_use(){
  import_data coupon_use "select
                          id,
                          coupon_id,
                          user_id,
                          order_id,
                          coupon_status,
                          get_time,
                          using_time,
                          used_time
                        from coupon_use
                        where (date_format(get_time,'%Y-%m-%d')='$do_date'
                        or date_format(using_time,'%Y-%m-%d')='$do_date'
                        or date_format(used_time,'%Y-%m-%d')='$do_date')"
}

import_order_status_log(){
  import_data order_status_log "select
                                  id,
                                  order_id,
                                  order_status,
                                  operate_time
                                from order_status_log
                                where date_format(operate_time,'%Y-%m-%d')='$do_date'"
}

import_activity_order(){
  import_data activity_order "select
                                id,
                                activity_id,
                                order_id,
                                create_time
                              from activity_order
                              where date_format(create_time,'%Y-%m-%d')='$do_date'"
}

import_user_info(){
  import_data "user_info" "select 
                            id,
                            name,
                            birthday,
                            gender,
                            email,
                            user_level, 
                            create_time,
                            operate_time
                          from user_info 
                          where (DATE_FORMAT(create_time,'%Y-%m-%d')='$do_date' 
                          or DATE_FORMAT(operate_time,'%Y-%m-%d')='$do_date')"
}

import_order_detail(){
  import_data order_detail "select 
                              od.id,
                              order_id, 
                              user_id, 
                              sku_id,
                              sku_name,
                              order_price,
                              sku_num, 
                              od.create_time,
                              source_type,
                              source_id  
                            from order_detail od
                            join order_info oi
                            on od.order_id=oi.id
                            where DATE_FORMAT(od.create_time,'%Y-%m-%d')='$do_date'"
}

import_payment_info(){
  import_data "payment_info"  "select 
                                id,  
                                out_trade_no, 
                                order_id, 
                                user_id, 
                                alipay_trade_no, 
                                total_amount,  
                                subject, 
                                payment_type, 
                                payment_time 
                              from payment_info 
                              where DATE_FORMAT(payment_time,'%Y-%m-%d')='$do_date'"
}

import_comment_info(){
  import_data comment_info "select
                              id,
                              user_id,
                              sku_id,
                              spu_id,
                              order_id,
                              appraise,
                              comment_txt,
                              create_time
                            from comment_info
                            where date_format(create_time,'%Y-%m-%d')='$do_date'"
}

import_order_refund_info(){
  import_data order_refund_info "select
                                id,
                                user_id,
                                order_id,
                                sku_id,
                                refund_type,
                                refund_num,
                                refund_amount,
                                refund_reason_type,
                                create_time
                              from order_refund_info
                              where date_format(create_time,'%Y-%m-%d')='$do_date'"
}

import_sku_info(){
  import_data sku_info "select 
                          id,
                          spu_id,
                          price,
                          sku_name,
                          sku_desc,
                          weight,
                          tm_id,
                          category3_id,
                          create_time
                        from sku_info where 1=1"
}

import_base_category1(){
  import_data "base_category1" "select 
                                  id,
                                  name 
                                from base_category1 where 1=1"
}

import_base_category2(){
  import_data "base_category2" "select
                                  id,
                                  name,
                                  category1_id 
                                from base_category2 where 1=1"
}

import_base_category3(){
  import_data "base_category3" "select
                                  id,
                                  name,
                                  category2_id
                                from base_category3 where 1=1"
}

import_base_province(){
  import_data base_province "select
                              id,
                              name,
                              region_id,
                              area_code,
                              iso_code
                            from base_province
                            where 1=1"
}

import_base_region(){
  import_data base_region "select
                              id,
                              region_name
                            from base_region
                            where 1=1"
}

import_base_trademark(){
  import_data base_trademark "select
                                tm_id,
                                tm_name
                              from base_trademark
                              where 1=1"
}

import_spu_info(){
  import_data spu_info "select
                            id,
                            spu_name,
                            category3_id,
                            tm_id
                          from spu_info
                          where 1=1"
}

import_favor_info(){
  import_data favor_info "select
                          id,
                          user_id,
                          sku_id,
                          spu_id,
                          is_cancel,
                          create_time,
                          cancel_time
                        from favor_info
                        where 1=1"
}

import_cart_info(){
  import_data cart_info "select
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
                      from cart_info
                      where 1=1"
}

import_coupon_info(){
  import_data coupon_info "select
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
                        from coupon_info
                        where 1=1"
}

import_activity_info(){
  import_data activity_info "select
                              id,
                              activity_name,
                              activity_type,
                              start_time,
                              end_time,
                              create_time
                            from activity_info
                            where 1=1"
}

import_activity_rule(){
    import_data activity_rule "select
                                    id,
                                    activity_id,
                                    condition_amount,
                                    condition_num,
                                    benefit_amount,
                                    benefit_discount,
                                    benefit_level
                                from activity_rule
                                where 1=1"
}

import_base_dic(){
    import_data base_dic "select
                            dic_code,
                            dic_name,
                            parent_code,
                            create_time,
                            operate_time
                          from base_dic
                          where 1=1"
}

case $1 in
  "order_info")
     import_order_info
;;
  "base_category1")
     import_base_category1
;;
  "base_category2")
     import_base_category2
;;
  "base_category3")
     import_base_category3
;;
  "order_detail")
     import_order_detail
;;
  "sku_info")
     import_sku_info
;;
  "user_info")
     import_user_info
;;
  "payment_info")
     import_payment_info
;;
  "base_province")
     import_base_province
;;
  "base_region")
     import_base_region
;;
  "base_trademark")
     import_base_trademark
;;
  "activity_info")
      import_activity_info
;;
  "activity_order")
      import_activity_order
;;
  "cart_info")
      import_cart_info
;;
  "comment_info")
      import_comment_info
;;
  "coupon_info")
      import_coupon_info
;;
  "coupon_use")
      import_coupon_use
;;
  "favor_info")
      import_favor_info
;;
  "order_refund_info")
      import_order_refund_info
;;
  "order_status_log")
      import_order_status_log
;;
  "spu_info")
      import_spu_info
;;
  "activity_rule")
      import_activity_rule
;;
  "base_dic")
      import_base_dic
;;
# 这里first是只在第一次导入的时候导入地区和省份表，因为后面两张表不会怎么变。然后从第二天开始每次重复导入的表不再有地区和省份表
"first")
   import_base_category1
   import_base_category2
   import_base_category3
   import_order_info
   import_order_detail
   import_sku_info
   import_user_info
   import_payment_info
   import_base_province
   import_base_region
   import_base_trademark
   import_activity_info
   import_activity_order
   import_cart_info
   import_comment_info
   import_coupon_use
   import_coupon_info
   import_favor_info
   import_order_refund_info
   import_order_status_log
   import_spu_info
   import_activity_rule
   import_base_dic
;;
"all")
   import_base_category1
   import_base_category2
   import_base_category3
   import_order_info
   import_order_detail
   import_sku_info
   import_user_info
   import_payment_info
   import_base_trademark
   import_activity_info
   import_activity_order
   import_cart_info
   import_comment_info
   import_coupon_use
   import_coupon_info
   import_favor_info
   import_order_refund_info
   import_order_status_log
   import_spu_info
   import_activity_rule
   import_base_dic
;;
esac
```

说明：脚本没有很好的运用MR的并发能力，因为执行的每行import命令并没有依赖关系，所以我们可以将脚本分为多个，一个脚本里写三个或多个import命令，Azkaban可以识别多个脚本的依赖关系来运行脚本。

导入数据后如果发现有lzo文件，但是没有索引，说明lzo压缩文件是空的，空的就没必要建立索引，这时查看下是否是日期设置错了，过滤不到要求的日期。

对脚本进行优化，将相同对象的表放在一起脚本里，将原来一个脚本分为6个并发执行，提高了MR的效率。

```bash
[dw@hadoop116 mysql_to_hdfs]$ ls
mysql_activity_2_hdfs.sh  mysql_orderp_2_hdfs.sh
mysql_base_2_hdfs.sh      mysql_products_2_hdfs.sh
mysql_coupon_2_hdfs.sh    mysql_user_relate_2_hdfs.sh


[dw@hadoop116 mysql_to_hdfs]$ ./mysql_activity_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1
 
[dw@hadoop116 mysql_to_hdfs]$ ./mysql_orderp_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1 &
[1] 29019
[dw@hadoop116 mysql_to_hdfs]$ ./mysql_base_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1 &
[2] 29021
[1]   Exit 2                  ./mysql_orderp_2_hdfs.sh all 2020-06-15 > /dev/null 2>&1
[dw@hadoop116 mysql_to_hdfs]$ ./mysql_products_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1 &
[3] 29251
[dw@hadoop116 mysql_to_hdfs]$ ./mysql_coupon_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1 &
[4] 29550
[dw@hadoop116 mysql_to_hdfs]$ ./mysql_user_relate_2_hdfs.sh all 2020-06-15 >/dev/null 2>&1 &
[5] 29890

```

最后在对应路径下查看数据结果：

```bash
[dw@hadoop116 mysql_to_hdfs]$ hadoop fs -ls /origin_data/gmall/db/activity_info
Found 2 items
drwxr-xr-x   - dw supergroup          0 2021-07-04 17:22 /origin_data/gmall/db/activity_info/2020-06-15
[dw@hadoop116 mysql_to_hdfs]$ hadoop fs -ls /origin_data/gmall/db/user_info
Found 2 items
drwxr-xr-x   - dw supergroup          0 2021-07-04 16:07 /origin_data/gmall/db/user_info/2020-06-15
```



## 数据同步策略

Mysql的数据每天都会新增或修改发生变化，对应着数仓的数据也应该同步发生变化才能保证分析结果的准确性，因此需要考虑怎样的同步策略能够保证MySQL数据和数仓数据能够同步变化。

目前业务数据都存放在mysl中，既然数据都在mysql中，mysql也支持sql查询，那为啥不写sql进行数据查询？因为：

1. 大数据量的分析会影响业务系统；
2. MySQL中没有历史数据(所以会把每天MySQL的数据导入到Hive当中，导入到HDFS里，后面分析直接走大数据集群，不会影响业务系统)；

> Sqoop的import和export
> Import支持将mysql表导入到hive表里或hbase表里或者hdfs的一个路径。
> Export只支持将HDFS下的一个路径文件导入到mysql的一张表里。

**数据同步策略的类型包括：全量同步、增量同步、新增及变化同步、特殊情况**

- **全量表：**存储完整的数据。（全量同步适用于即会发生新增又会发生变化的表，比如说用户表，将mysql的全表数据导一份，放到数仓当中。全量表也是分区表，一天一分区，存放的是当天从mysql导入进来的全量数据，由于数仓有一个特点需要保留历史数据，但并不是一直保留，一般保留个半年。）
- **增量表：**存储新增加的数据。（mysql新增的数据用hive的增量表进行存储，增量表是分区表，按天分区，mysql当天新增数据就导入放在hive中当天的分区中，后面类似。这样的表在Mysql中要求每天只有新增，没有变化的数据，一般流水表就是这种表，比如支付流水表，只会新增不会修改，再比如订单状态表。）
- **新增及变化表：**存储新增加的数据和变化的数据。（Mysql数据表很大，每天有新增有变化，如果还是采用每日全量的策略不太好，冗余太大，效率低。这时采用新增及变化同步，每天只把修改的数据和新增的数据拿过来导入数仓，不变的数据不导入。对于这种表，要不就是获取全量最新的数据，要不就是获取历史上某一天的全量数据。对于新增及变化表，就不好获取了，后面数仓部分再进行解释）
- **特殊表：**只需要存储一次。



### 全量同步策略

每日全量，就是每日存储一份完整数据作为一个分区。适用于表数据量不大，且每天既会有新数据插入，也会有旧数据的修改的场景。

例如：编码字典表、品牌表、商品三级分类、商品二级分类、商品一级分类、优惠规则表、活动表、SKU商品表、SPU商品表等。



![img](https://pic2.zhimg.com/80/v2-06d9c5d0211f77880010766aaa4b44e9_1440w.webp)





### 增量同步策略

每日增量，就是每天存储一份增量数据，作为一个分区。适用于表数据量大，且每日按只会有新数据插入的场景。例如：退单表、订单状态表、支付流水表、订单详情表、活动与订单关联表、商品评论表。



![img](https://pic1.zhimg.com/80/v2-b3b0969c826e47c2fe430ecc313b7a98_1440w.webp)





### 新增及变化策略

每日新增变化，就是存储创建时间和操作时间都是今天的数据，适用于表的数据量大，既会有新增，有会有变化。例如：用户表、订单表、优惠卷领用表。



![img](https://pic4.zhimg.com/80/v2-96c0d8231725d42cd1db8eb94669643b_1440w.webp)





### 特殊策略

不管是新增还是全量还是新增及变化策略，都需要每天到mysql导入数据，但有些数据不需要每天都去同步，比如省份、地区数据，只需在数仓保留一份即可。

某些特殊的维度表，可不必遵循上述同步策略。

1. 客观世界维度

没变化的客观世界的维度（比如性别，地区，民族，政治成分，鞋子尺码）可以只存一份固定值。

1. 日期维度

日期维度可以一次性导入一年或若干年的数据。保留历史数据能够分析数据和时间维度的关系。后期会建立一个时间维度表。日期维度表主键是日期(20200916)，字段比如说，是这一年里的第几天，位于一年里第几个季度，今天周几，是否法定节假日等。



## 数仓搭建-ODS层

1. 保持数据原貌不做任何修改，起到备份数据的作用。
2. 数据采用LZO压缩，减少磁盘存储空间。100G数据可以压缩到10G以内。
3. 创建分区表，防止后续的全表扫描，在企业开发中大量使用分区表。
4. 创建外部表。在企业开发中，除了自己用的临时表，创建内部表外，绝大多数场景都是创建外部表。

### 创建日志表ods_log

ODS层以原始数据为基准，数据怎么样，表就怎么建，由于用户日志是json格式，所以我们只需要建立一张外部表（保护原始数据），一个字段，做好分区。



![img](https://pic2.zhimg.com/80/v2-55a9a2de8ffe8929f1eea60c2f8aa1ed_1440w.webp)



```sql
-- 建表
drop table if exists ods_log;

CREATE EXTERNAL TABLE ods_log(line string) -- 创建外部表避免影响原始数据
PARTITION BY (`dt` string) -- 按照日志时间创建分区表
STORED AS -- 设置存储格式(Hive存储和读取时用到) 读数据采LzoTextInputFormat；
    INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
    OUTPUTFORMAT 'org.apache.hadoop.hive.sql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION '/warehouse/gmall/ods/ods_log'; -- 指定数据路径

-- 加载数据
LOAD DATA INPATH '/origin_date/gmall/log/topic_log/2020-06-15' INTO TABLE ods_log PARTITION (dt='2020-06-15');

# 当写sql读取表数据时，因为导入文件用lzo压缩所以使用指定INPUTFORMAT；
# 当写sql写入表数据时，因为导入文件用lzo压缩所以使用指定OUTPUTFORMAT
```

> 注意：时间格式都配置成YYYY-MM-DD格式，这是Hive默认支持的时间格式

编写日志数据导入脚本：

```text
#! /bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive
hadoop=/opt/module/hadoop-3.1.3/bin/hadoop

# 申明时间变量，每天执行前一天的脚本，一般在凌晨之后
if [ -n "$1" ] ;then
   do_date=$1
else
   do_date=`date -d "-1 day" +%F`
fi

echo ================== 日志日期为 $do_date ==================

# 字符串最外面的引号是双引，可以解析里面的变量$do_date
sql = "
LOAD DATA INPATH '/origin_date/gmall/log/topic_log/$do_date' INTO TABLE ${APP}.ods_log PARTITION(dt='$do_date');
"
# “{}”将变量名和其他字符串分割开来

# 这里要注意，一定要加上引号，因为hive -e执行的是sql整体，单一个$sql会传入一长串sql，hive识别不了
$hive -e "$sql"

# 为数据建立lzo索引
$hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/common/hadoop-lzo-0.4.20.jar com.hadoop.compression.lzo.DistributedLzoIndexer -Dmapreduce.job.queuename=hive /warehouse/$APP/ods/ods_log/dt=$do_date
```

> 注意：[ -n 变量值 ]不会解析数据，使用[ -n 变量值 ]时，需要对变量加上双引号(" ")

给予脚本可执行权限，执行脚本：

```text
[dw@hadoop116 mysql_to_hdfs]$ hdfs_to_ods_log.sh 2020-06-15
================== 日志日期为 2020-06-15 ==================
Hive Session ID = 1cff6edc-879c-4990-b0a1-ffacccff233c
Loading data to table gmall.ods_log partition (dt=2020-06-15)
OK
Time taken: 3.016 seconds
```

这里要注意，执行了数据导入命令后，原来存日志数据的路径下的日志数据会转移到数据仓库下的ODS路径。



![img](https://pic3.zhimg.com/80/v2-fcf0deef2d8ae2f0bffda79e0cc45e2a_1440w.webp)



**补充：**Shell中单引号和双引号区别

编写一个sh脚本测试：

```bash
#!/bin/bash
do_date=$1

echo '$do_date'
echo "$do_date"
echo "'$do_date'"
echo '"$do_date"'
echo `date`
```

执行结果：

```bash
[atguigu@hadoop102 bin]$ test.sh 2020-06-14
$do_date
2020-06-14
'2020-06-14'
"$do_date"
2020年 06月 18日 星期四 21:02:08 CST
```

**总结：**

- 单引号不取变量值
- 双引号取变量值
- 反引号`，执行引号中命令
- 双引号内部嵌套单引号，取出变量值
- 单引号内部嵌套双引号，不取出变量值



### ODS层业务数据



![img](https://pic2.zhimg.com/80/v2-c8589cb4a5cec9fedbdddb843b2bfe71_1440w.webp)





![img](https://pic4.zhimg.com/80/v2-f9fc29ace6bceb25826d858f308cf35f_1440w.webp)



### 订单表（增量及更新）

建表语句：

```sql
drop table if exists ods_order_info;
create external table ods_order_info (
    `id` string COMMENT '订单号',
    `final_total_amount` decimal(16,2) COMMENT '订单金额',
    `order_status` string COMMENT '订单状态',
    `user_id` string COMMENT '用户id',
    `out_trade_no` string COMMENT '支付流水号',
    `create_time` string COMMENT '创建时间',
    `operate_time` string COMMENT '操作时间',
    `province_id` string COMMENT '省份ID',
    `benefit_reduce_amount` decimal(16,2) COMMENT '优惠金额',
    `original_total_amount` decimal(16,2)  COMMENT '原价金额',
    `feight_fee` decimal(16,2)  COMMENT '运费'
) COMMENT '订单表'
PARTITIONED BY (`dt` string) -- 按照时间创建分区
row format delimited fields terminated by '\t' -- 指定分割符为\t 
STORED AS -- 指定存储方式，读数据采用LzoTextInputFormat；输出数据采用TextOutputFormat
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_order_info/' -- 指定数据在hdfs上的存储位置
;
```

### 订单详情表(增量表)

```sql
drop table if exists ods_order_detail;
create external table ods_order_detail( 
    `id` string COMMENT '编号',
    `order_id` string  COMMENT '订单号', 
    `user_id` string COMMENT '用户id',
    `sku_id` string COMMENT '商品id',
    `sku_name` string COMMENT '商品名称',
    `order_price` decimal(16,2) COMMENT '商品价格',
    `sku_num` bigint COMMENT '商品数量',
    `create_time` string COMMENT '创建时间',
    `source_type` string COMMENT '来源类型',
    `source_id` string COMMENT '来源编号'
) COMMENT '订单详情表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t' 
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_order_detail/';
```

### SKU商品表(全量表)

```sql
drop table if exists ods_sku_info;
create external table ods_sku_info( 
    `id` string COMMENT 'skuId',
    `spu_id` string   COMMENT 'spuid', 
    `price` decimal(16,2) COMMENT '价格',
    `sku_name` string COMMENT '商品名称',
    `sku_desc` string COMMENT '商品描述',
    `weight` string COMMENT '重量',
    `tm_id` string COMMENT '品牌id',
    `category3_id` string COMMENT '品类id',
    `create_time` string COMMENT '创建时间'
) COMMENT 'SKU商品表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_sku_info/';
```

### 用户表(增量及更新)

```sql
drop table if exists ods_user_info;
create external table ods_user_info( 
    `id` string COMMENT '用户id',
    `name`  string COMMENT '姓名',
    `birthday` string COMMENT '生日',
    `gender` string COMMENT '性别',
    `email` string COMMENT '邮箱',
    `user_level` string COMMENT '用户等级',
    `create_time` string COMMENT '创建时间',
    `operate_time` string COMMENT '操作时间'
) COMMENT '用户表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_user_info/';
```

### 商品一级分类表(全量)

```sql
drop table if exists ods_base_category1;
create external table ods_base_category1( 
    `id` string COMMENT 'id',
    `name`  string COMMENT '名称'
) COMMENT '商品一级分类表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_base_category1/';
```

### 商品二级分类表（全量）

```sql
drop table if exists ods_base_category2;
create external table ods_base_category2( 
    `id` string COMMENT ' id',
    `name` string COMMENT '名称',
    category1_id string COMMENT '一级品类id'
) COMMENT '商品二级分类表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_base_category2/';
```

### 支付流水表（增量）

```sql
drop table if exists ods_payment_info;
create external table ods_payment_info(
    `id`   bigint COMMENT '编号',
    `out_trade_no`    string COMMENT '对外业务编号',
    `order_id`        string COMMENT '订单编号',
    `user_id`         string COMMENT '用户编号',
    `alipay_trade_no` string COMMENT '支付宝交易流水编号',
    `total_amount`    decimal(16,2) COMMENT '支付金额',
    `subject`         string COMMENT '交易内容',
    `payment_type`    string COMMENT '支付类型',
    `payment_time`    string COMMENT '支付时间'
)  COMMENT '支付流水表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_payment_info/';
```

### 省份表（特殊）

```sql
drop table if exists ods_base_province;
create external table ods_base_province (
    `id`   bigint COMMENT '编号',
    `name`        string COMMENT '省份名称',
    `region_id`    string COMMENT '地区ID',
    `area_code`    string COMMENT '地区编码',
    `iso_code` string COMMENT 'iso编码,superset可视化使用'
)  COMMENT '省份表'
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_base_province/';
```

### 地区表（特殊）

```sql
drop table if exists ods_base_region;
create external table ods_base_region (
    `id` string COMMENT '编号',
    `region_name` string COMMENT '地区名称'
)  COMMENT '地区表'
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_base_region/';
```

### 品牌表（全量）

```sql
drop table if exists ods_base_trademark;
create external table ods_base_trademark (
    `tm_id`   string COMMENT '编号',
    `tm_name` string COMMENT '品牌名称'
)  COMMENT '品牌表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_base_trademark/';
```

### 订单状态表（增量）

```sql
drop table if exists ods_order_status_log;
create external table ods_order_status_log (
    `id`   string COMMENT '编号',
    `order_id` string COMMENT '订单ID',
    `order_status` string COMMENT '订单状态',
    `operate_time` string COMMENT '修改时间'
)  COMMENT '订单状态表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_order_status_log/';
```

### SPU商品表（全量）

```sql
drop table if exists ods_spu_info;
create external table ods_spu_info(
    `id` string COMMENT 'spuid',
    `spu_name` string COMMENT 'spu名称',
    `category3_id` string COMMENT '品类id',
    `tm_id` string COMMENT '品牌id'
) COMMENT 'SPU商品表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_spu_info/';
```

### 商品评论表（增量）

```sql
drop table if exists ods_comment_info;
create external table ods_comment_info(
    `id` string COMMENT '编号',
    `user_id` string COMMENT '用户ID',
    `sku_id` string COMMENT '商品sku',
    `spu_id` string COMMENT '商品spu',
    `order_id` string COMMENT '订单ID',
    `appraise` string COMMENT '评价',
    `create_time` string COMMENT '评价时间'
) COMMENT '商品评论表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_comment_info/';
```

### 退单表（增量）

```sql
drop table if exists ods_order_refund_info;
create external table ods_order_refund_info(
    `id` string COMMENT '编号',
    `user_id` string COMMENT '用户ID',
    `order_id` string COMMENT '订单ID',
    `sku_id` string COMMENT '商品ID',
    `refund_type` string COMMENT '退款类型',
    `refund_num` bigint COMMENT '退款件数',
    `refund_amount` decimal(16,2) COMMENT '退款金额',
    `refund_reason_type` string COMMENT '退款原因类型',
    `create_time` string COMMENT '退款时间'
) COMMENT '退单表'
PARTITIONED BY (`dt` string)
row format delimited fields terminated by '\t'
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
location '/warehouse/gmall/ods/ods_order_refund_info/';
```