- [DataX脚本任务开发记录](https://blog.csdn.net/sosemseo/article/details/118941071)

# 1 DataX脚本任务开发记录

## 1.1 每小时同步一次，每次拉近9天数据

```bash
#!/bin/bash
# 每小时同步一次，每次拉近9天数据
# 0 */1 * * * (cd /home/work/daiyuanpei; bash t_test_days.sh 9 > /home/work/daiyuanpei/t_test_days.out 2>&1 &)
. ~/.bashrc
dateEnd=`date +"%Y%m%d"`
# 默认开始时间30天前
dateBegin=`date +%Y%m%d -d"-30 day $dateEnd"`
# 判断是否传参并校验是否为数字、是否大于100
if [ $# == 1 ]; then
    if [ ! -n "$(echo $1| sed -n "/^[0-9]\+$/p")" ]; then
        echo "FATAL: The parameter is not a number."
        exit -1
    fi
    if [ $1 -gt 100 ]; then
        echo "FATAL: Parameter n cannot be greater than 100."
        exit -1
    fi
    dateBegin=`date +%Y%m%d -d"-$1 day $dateEnd"`
fi
delDateBegin=`date +%Y-%m-%d -d"+1 day $dateBegin"`
echo $dateBegin $dateEnd $delDateBegin
schema="fn"
tbname="t_test"
example="/home/work/datax/job/fn/example/${schema}_${tbname}_days"
tempdir="/home/work/datax/job/fn"
basepath=$(cd `dirname $0`; pwd)
cd $basepath
jsonfile="${tempdir}/${schema}_${tbname}_days"
if [[ "$dateBegin" > "$dateEnd" ]]; then
    echo "FATAL: dateEnd must be greater than or equal to dateBegin. dateBegin: $dateBegin, dateEnd: $dateEnd"
    exit -1
fi
jdbcTpl="{\"jdbcUrl\":[\"jdbc:mysql://*.*.*.*:1234/dyp_db_DBDATE\"],\"table\":[\"t_test\"]}"
jdbcStr=""
while [[ "$dateBegin" < "$dateEnd" ]] || [[ "$dateBegin" == "$dateEnd" ]]; do
    dayStr="${jdbcTpl/DBDATE/$dateBegin}"
    jdbcStr="$jdbcStr,$dayStr"
    dateBegin=`date +%Y%m%d -d"+1 day $dateBegin"`
done
echo "while done"
# 去掉第一个字符
jdbcStr=`echo $jdbcStr | awk '{print substr($1,2)}'`
# 下面sed 所以需要转义斜线和双引号
# 单斜线替换一次 双斜线替换所有匹配
jdbcStr="${jdbcStr//\"/\\\"}"
jdbcStr="${jdbcStr//\//\/}"
echo $jdbcStr
sed "s/JDBCSTR/${jdbcStr}/g;s/CURRENTDATE/${dateEnd}/g" ${example} > ${jsonfile}
echo "${jsonfile} done"
~/datax/bin/start_datax.sh ${jsonfile} && bash /home/work/datax/sh-load/t_test_days.sh ${schema} ${tbname} ${delDateBegin}
echo "*****************************"
echo "*****************************"
echo "*****************************"
```



```bash
#!/bin/bash
# /home/work/datax/sh-load/t_test_days.sh
. ~/.bashrc
if [ $# -lt 3 ]
then
    echo "./loaddata schema tablename delDateBegin"
    exit
fi
export LANG="zh_CN.UTF-8"
export LC_ALL="zh_CN.UTF-8"
schema=$1
tablename=$2
delDateBegin=$3
data_path=" /home/work/dyp-datax-admin/ext/r_ext_${schema}_${tablename}_old.dat"
basepath=$(cd `dirname $0`; pwd)
cd ${basepath}
if [ ! -e ${data_path} ];then
 echo "no exist filepath:${data_path}"
 exit
fi
 
#DEL_SQL="delete from ${schema}.${tablename} where dt>='${delDateBegin}';"
LOAD_SQL="delete from ${schema}.${tablename} where dt>='${delDateBegin}';insert into ${schema}.${tablename} select * from pay_data_ext.r_ext_${schema}_${tablename}_old;"
#echo ${DEL_SQL}
echo ${LOAD_SQL}
tempfile="${basepath}/temp.log"
export PGPASSWORD=******
/usr/bin/psql -h *.*.*.* -p 5432 -d dyp_gp_pay -U gp_user -c "$LOAD_SQL" > ${tempfile} 2>&1
if [ $? -eq 0 ]
   then
        logmsg=`cat ${tempfile}`
        echo "load to success ${tablename}:${logmsg}"
        emailTitle="[云上][${schema}.${tablename}] load success"
        emailContent=`echo "datax load to gppay success: ${schema}.${tablename}: ${logmsg}"`
     else
        logmsg=`cat ${tempfile}`
        msg=`echo "datax load to gppay failed:${tablename}:${logmsg}"`
        emailTitle="[云上][${schema}.${tablename}] load failed"
        emailContent=`echo "datax load to gppay failed: ${schema}.${tablename}: ${logmsg}"`
        time=$(date "+%Y-%m-%d %H:%M:%S")
        echo "${time} ${schema}.${tablename} FATAL" >> /home/work/datax/sh-load/log/datax.log.wf
fi
echo ${emailContent} | mail -s "${emailTitle}" daiyuanpei@daiyuanpei.com
```

## 1.2 datax全流程脚本

- [datax全流程脚本](https://blog.csdn.net/sosemseo/article/details/104803624)

```bash
#!/bin/bash
# ~/datax/bin/start_datax.sh
 
. ~/.bashrc
echo $#
if [ $# -lt 1 ]
then
    echo "need job config file"
    exit
fi
jobname="$1"
taskname=`echo $(basename $jobname .xml)`
 
echo "$jobname,taskname=$taskname"
DATAX_HOME=/home/work/datax
timenow=` date +'%H%M%S'`
cd $DATAX_HOME
outputfile="${DATAX_HOME}/log/${taskname}.log"
errorfile="${DATAX_HOME}/log/${taskname}.error"
rm ${outputfile} -f
python ${DATAX_HOME}/bin/datax.py ${jobname} >> ${outputfile} 2>${errorfile}
if [ $? -eq 0 ]
then
    tail  -n32 ${outputfile}
    exit 0
else
    tail  -n32 ${outputfile}
    exit 102
fi
tail  -n32 ${outputfile}
exit 103
```



```bash
#!/bin/bash
# core_incr.sh
. ~/.bashrc
start_date=`date -d "1 days ago" +%Y-%m-%d`
if [ $# -gt 0  ]
then
    start_date=$1
fi
end_date=`date -d "${start_date} 1 days " "+%Y-%m-%d"`
schema="core"
tbname="t_trans"
echo ${start_date} ${end_date}
 
example="/home/work/datax/job/core/example"
tempdir="/home/work/datax/job/core"
basepath=$(cd `dirname $0`; pwd)
cd $basepath
ARR=(1 2 3 4 5 6 7 8)
for ID in ${ARR[*]}
do
    jsonfile="${tempdir}/${schema}_${tbname}_$ID"
    oneexample="${example}/t_trans_example_$ID"
    sed "s/START_DATE/${start_date}/g;s/END_DATE/${end_date}/g" ${oneexample} > ${jsonfile}
done
echo "${jsonfile} done"
ARR=(1 2 3 4 5 6 7 8)
for ID in ${ARR[*]}
do
    echo "~/datax/log/${schema}_${tbname}_$ID.log"
    ~/datax/bin/start_datax.sh /home/work/datax/job/core/${schema}_${tbname}_$ID > /home/work/daiyuanpei/core_incr_$ID.out && bash /home/work/datax/job/dataxload_core.sh core t_trans p$ID >> /home/work/daiyuanpei/core_incr_$ID.out 2>&1 &
    sleep 3m
done
echo "core_incr.sh done"
```



```bash
#!/bin/bash
# /home/work/daiyuanpei/retry_dataxload_core.sh
. ~/.bashrc
if [ $# -lt 3 ]
then
    echo "./loaddata schema tablename hostindex"
    exit
fi
export LANG="zh_CN.UTF-8"
export LC_ALL="zh_CN.UTF-8"
schema=$1
tablename=$2
hostindex=$3
data_path=" /home/work/datax/ext/core/${tablename}_${hostindex}"
basepath=$(cd `dirname $0`; pwd)
cd ${basepath}
if [ ! -e ${data_path} ];then
 echo "no exist filepath:${data_path}"
 exit
fi
 
LOAD_SQL="insert into ${schema}.${tablename} select to_date(F_create_time::text,'YYYY-MM-DD'),* from pay_data_ext.r_ext_${tablename}_${hostindex};"
 
tempfile="${basepath}/temp.log"
export PGPASSWORD=************
/usr/bin/psql -h **.***.**.*** -p 5432 -d qq_gp_db -U gp_rd -c "$LOAD_SQL" > ${tempfile} 2>&1
if [ $? -eq 0 ]
   then
        logmsg=`cat ${tempfile}`
        echo "load to success ${tablename}:${logmsg}"
        emailTitle="[重试][${schema}.${tablename}_${hostindex}] load success"
        emailContent=`echo "datax load to gppay success: ${schema}.${tablename}_${hostindex}: ${logmsg}"`
     else
        logmsg=`cat ${tempfile}`
        msg=`echo "datax load to gppay failed:${tablename}:${logmsg}"`
        emailTitle="[重试][${schema}.${tablename}_${hostindex}] load failed"
        emailContent=`echo "datax load to gppay failed: ${schema}.${tablename}_${hostindex}: ${logmsg}"`
        # 失败创建表名空文件
	    touch /home/work/daiyuanpei/monitor/${tablename}
        # 失败写日志
        time=$(date "+%Y-%m-%d %H:%M:%S")
        echo "${time} ${schema}.${tablename} FATAL" >> /home/work/datax/sh-load/log/datax.log.wf
fi
echo ${emailContent} | mail -s "${emailTitle}" daiyp@qq.com -c "daiyuanpei@qq.com daiyuankai@qq.com"
```



```bash
#!/bin/bash
# monitor.sh 推送文件到开发机监控目录 当时用的这个方法做监控
# 3 8 * * * (cd /home/work/daiyuanpei; bash ./monitor.sh > ct_monitor.out 2>&1 &)
. ~/.bashrc
 
current_date=`date +%Y-%m-%d`
echo ${current_date}
basepath=$(cd `dirname $0`; pwd)
cd $basepath
BAK=/home/work/daiyuanpei/monitor
if [ "`ls -A ${BAK}`" = "" ];
then
	echo "${BAK} is empty"
else
    echo "${BAK} is not empty"
	/usr/local/bin/scp -i /home/work/.ssh/id_rsa /home/work/daiyuanpei/monitor/t_* work@**.***.**.***:/home/work/daiyuanpei/monitor
	echo "file move done"
	rm ./monitor/t_*
	echo "empty folder done"
fi
```

# 2 Datax生产脚本配置

- [datax介绍及生产脚本配置](https://blog.csdn.net/XDSXHDYY/article/details/103993991)

栗子：读取sqlserver写入hdfs （再将hdfs数据加载到hive表），总任务执行脚本execute.sh。

```bash
sh execute.sh table_name start_time end_time
```

一共可以传三个参数

table_name：你要执行同步的表名，多个表用逗号分隔开（比如"table1,table2"）

start_time,end_time：同步数据的时间段，这个时间段在每张表的where条件里面用时间戳字段来过滤

如果你提前确定了表名就可以在脚本里面配好，时间段默认是前一天。

这样就不用传参数了直接执行脚本: sh execute.sh (记得要提前在hive里面建好对应表，导完数据会load进hive表)

```bash
#!/bin/bash
source /etc/profile
#数据库连接信息
connstr="jdbc:sqlserver://11.99.99.99:1433;DatabaseName=databs"
username="root"
password="123456"
#源数据库表
#名获取开始和结束的日期
if [ $# -gt 1 ]
then
    table_name=$1
    start_time=$2
    end_time=$3
else
    table_name="table1,table2,table3,table4"
    start_time=`date -d "-1 day" +%Y-%m-%d`
    end_time=`date +%Y-%m-%d`
fi

#hdfs地址信息
defaultFS="hdfs://cdh03:8020"
#datax导入文件后的存储格式
fileType="text"
#写入hdfs的模式
writeMode="append"
#文件数据分隔符
separator='\t'
#hive库
database="ods_data_all"

array=(${table_name//,/ })

for var in ${array[@]}
do

  #datax数据导入目录
  datadir="/data/datax/databs/${var}/${start_time}/"
  #datax数据导入存储的文件名
  fileName='$start_time'
  #创建数据导入目录命令
  hdfs dfs -mkdir -p $datadir
  #运行datax的命令
  python /opt/datax/bin/datax.py -p "-Dconnstr='$connstr' -Dusername='$username' -Dpassword='$password' -Dtable_name='$var' -Dstart_time='$start_time' -Dend_time='$end_time' -DdefaultFS='$defaultFS' -DfileType='$fileType' -Ddatadir='$datadir' -DfileName='$fileName' -DwriteMode='$writeMode' -Dseparator='$separator'" --jvm="-Xms1G -Xmx1G" /opt/datax/job/${var}.json
  #将数据加载进hive表
  hive -e "use ${database};load data inpath '${defaultFS}/${datadir}/*' into table ods_${var}_all;"
done
```

单个表的json配置文件：table1.json

```json
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "sqlserverreader",
                    "parameter": {
                        "connection": [
                            {
                                "jdbcUrl": ["$connstr"],
                                "querySql": ["select * from $table_name where creationtime >= '${start_time}' and creationtime < '${end_time}'"]
                            }
                        ],
                        "password": "$password",
                        "username": "$username"
                    }
                },
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "column": [
                                 {"name": "Id","type": "bigint"},
                                 {"name": "PkId","type": "int"},
                                 {"name": "Barcode","type": "String"},
                                 {"name": "ClassId","type": "int"},
                                 {"name": "Qty","type": "int"},
                                 {"name": "TerminalId","type": "int"},
                                 {"name": "Remark","type": "String"},
                                 {"name": "CreationTime","type": "TIMESTAMP"},
                                 {"name": "SaleClassId","type": "int"}
                                ],
                        "defaultFS": "$defaultFS",
                        "fieldDelimiter": "$separator",
                        "fileName": "$fileName",
                        "fileType": "$fileType",
                        "path": "$datadir",
                        "writeMode": "$writeMode"
                    }
                }
            }
        ],
        "setting": {
            "speed": {
                "channel": "3",
                "batchSize":2048
            },
            "errorLimit":{
                "percentage":0.02,
                "record":0
            }
        }
    }
}
```

自己再建一下其他表的json配置文件，主要每张表的字段不一样，其余配置都差不多。

# 3 Java执行Datax脚本-MySQL写入Doris

- [Java执行Datax脚本-MySQL写入Doris](https://blog.csdn.net/docsz/article/details/128906450)

## 1、生成DataX json模板

```java
public static void josnTemplate(String source,String target,String datasetId) {
    String classPath = "D:/software/datax";
    String jobFilePath = "D:/software/datax/job/";
    System.setProperty("datax.home", classPath);

    String[] arguments = new String[]{"python", classPath+"/bin/datax.py", "-r", source.toLowerCase()+"reader", "-w", target.toLowerCase()+"writer"};
    try {
        Process process = Runtime.getRuntime().exec(arguments);
        BufferedReader br = new BufferedReader(new InputStreamReader(process.getInputStream(), "UTF-8"));
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(
            new FileOutputStream(jobFilePath + "dataset_"+datasetId), "UTF-8"));

        String line = null;
        int count = 0;
        while ((line = br.readLine()) != null) {
            count++;
            if (count > 15) {
                //System.out.println(line);

                bw.write(line);
                bw.newLine();
                bw.flush();
            }
        }
        bw.close();
        br.close();
        //java代码中的process.waitFor()返回值为0表示我们调用python脚本成功，
        //返回值为1表示调用python脚本失败，这和我们通常意义上见到的0与1定义正好相反
        int re = process.waitFor();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 2、组装DataX json模板

```java
public static String getJson(String filePath) {
    String json = "";
    try {
        File file = new File(filePath);
        FileReader reader = new FileReader(file);
        FileInputStream inputStream = new FileInputStream(file);
        Reader stream = new InputStreamReader(inputStream, Charset.defaultCharset());

        int ch = 0;
        StringBuilder sb = new StringBuilder();
        while ((ch = reader.read()) != -1) {
            sb.append((char) ch);
        }
        reader.close();
        stream.close();

        JSONObject jobj = JSONObject.parseObject(sb.toString());
        json = jobj.toJSONString();
    } catch (Exception e) {
        e.printStackTrace();
    }
    return json;
}
```

```java
public static String getDataXJson(String filePath) throws FileNotFoundException {
    String json = getJson(filePath);
    JSONObject jobj = JSONObject.parseObject(json);
    //组装reader
    JSONObject reader = jobj.getJSONObject("job").getJSONArray("content").getJSONObject(0).getJSONObject("reader");
    JSONObject rp = reader.getJSONObject("parameter");
    rp.put("username", "root");
    rp.put("password", "123456");
    rp.remove("column");//删除column元素

    JSONObject rc = rp.getJSONArray("connection").getJSONObject(0);
    JSONArray jdbcUrl = new JSONArray();
    jdbcUrl.add("jdbc:mysql://IP:3306/dataset?useUnicode=true&characterEncoding=utf-8&useSSL=false&autoReconnect=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai");
    rc.put("jdbcUrl", jdbcUrl);
    rc.remove("table");//删除table元素
    JSONArray querySql = new JSONArray();
    querySql.add("select uuid() as fusion_uuid,a,b,c,d,e,f,h from t_number");
    rc.put("querySql", querySql);
    //组装writer
    JSONObject writer = jobj.getJSONObject("job").getJSONArray("content").getJSONObject(0).getJSONObject("writer");
    JSONObject wp = writer.getJSONObject("parameter");
    wp.put("username", "root");
    wp.put("password", "123456");
    JSONArray column = new JSONArray();
    column.add("*");
    wp.put("column", column);
    JSONArray preSql = new JSONArray();
    preSql.add("truncate table dataset_1619888763184439297");
    wp.put("preSql", preSql);
    JSONArray loadUrl = new JSONArray();
    loadUrl.add("IP:8030");
    wp.put("loadUrl", loadUrl);

    JSONObject wc = wp.getJSONArray("connection").getJSONObject(0);
    wc.put("jdbcUrl", "jdbc:mysql://IP:9030/fusion");
    JSONArray table = new JSONArray();
    table.add("dataset_1619888763184439297");
    wc.put("table", table);
    wc.put("selectedDatabase", "fusion");
    //组装setting
    JSONObject setting = jobj.getJSONObject("job").getJSONObject("setting");
    JSONObject speed = setting.getJSONObject("speed");
    speed.put("channel", "1");
    //组装errorLimit
    JSONObject errorlimit = new JSONObject();
    errorlimit.put("percentage",0.02);
    errorlimit.put("record",0);
    setting.put("errorLimit",errorlimit);
    //写入文本
    PrintStream stream= new PrintStream(filePath,"UTF-8");
    //stream.println(jobj);
    return jobj.toString();
}
```

## 3、执行DataX job

```java
public static void execute(String filePath) throws Exception {
    String dataxHome = "D:/software/datax";
    System.setProperty("datax.home", dataxHome);

    String dataXJson = getDataXJson(filePath);
    //PrintStream stream= new PrintStream(filePath);
    //stream.println(dataXJson);

    //String[] datxArgs = {"-job", getClasspath() + "/datax/stream.json", "-mode", "standalone", "-jobid", "-1"};
    String[] dataXArgs = {"-job", filePath, "-mode", "standalone", "-jobid", "-1"};

    try {
        Engine.entry(dataXArgs);
    }catch (DataXException e){
        e.printStackTrace();
    }
    catch (Throwable throwable) {
        /**
             * 拿到异常信息入库.
             */
        log.error("\n\n经DataX智能分析,该任务最可能的错误原因::\n" + ExceptionTracker.trace(throwable));
    }
}
```

```java
public static void main(String[] args) throws Exception {
    josnTemplate("mysql","doris","1619879625037537281");
    getDataXJson("D:/software/datax/job/dataset_1619879625037537281");
    execute("D:/software/datax/job/dataset_1619879625037537281");
}
```



# 附录

## A.1 定时任务相关问题

- [DataX踩坑2 | 定时任务crontab不执行或报错：/bin/sh: java: command not found](https://blog.csdn.net/qq_25112523/article/details/109333611)

crontab定时任务执行Shell脚本报错：`/bin/sh: java: command not found`

解决方法：在shell脚本中加入下面的参数即可：

```bash
. /etc/profile
```


加入后定时任务就可以正常的运行shell脚本了。
