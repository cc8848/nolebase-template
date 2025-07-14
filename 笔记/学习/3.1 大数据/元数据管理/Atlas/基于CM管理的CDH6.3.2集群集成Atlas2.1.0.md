- [基于CM管理的CDH6.3.2集群集成Atlas2.1.0](https://blog.csdn.net/AlierSnow/article/details/123730617)
- [搭建Atlas2.1.0 集成CDH6.3.2 生产环境版](https://blog.csdn.net/h952520296/article/details/110874432)
- [CDH6.3.0下Apache Atlas2.1.0安装与配置](https://www.cnblogs.com/abey-lu/p/14157403.html)
- [Atlas部署并集成HIVE_atlas 集成hive](https://blog.csdn.net/Yellow_python/article/details/122744861)
- [Atlas血缘分析在数据仓库中的实战案例](https://cloud.tencent.com/developer/article/1983831)
- [Apache Atlas 2.1.0编译打包安装记录](https://blog.csdn.net/qq_39945938/article/details/119004695)

## 1 基于CM管理的CDH6.3.2集群集成Atlas2.1.0

大数据平台进行数据治理需要，采用Apache Atlas进行数据治理。下载Atlas2.1.0版本源码包。下载https://www.apache.org/dyn/closer.cgi/atlas/2.1.0/apache-atlas-2.1.0-sources.tar.gz 到windows。

前提CDH集群已经搭建完成，组件服务包含Hdfs、Hive、Hbase、Solr、Kafka、Sqoop、Zookeeper、Impala、Yarn、Spark、Oozie、Phoenix、Hue等。

windows环境JDK（1.8.1_151以上）、Maven（3.5.0以上）版本最好和Linux中的JDK、Maven版本保持一致。

## 2 修改源码并编译

解压apache-[atlas](https://so.csdn.net/so/search?q=atlas&spm=1001.2101.3001.7020)-2.1.0-sources.tar.gz压缩包，得到apache-atlas-sources-2.1.0目录，用InetlliJ Idea打开项目文件，修改目录下的pom.xml文件。

在标签中修改组件为CDH中使用的版本。

```xml
<lucene-solr.version>7.4.0-cdh6.3.2</lucene-solr.version>
<hadoop.version>3.0.0-cdh6.3.2</hadoop.version>
<hbase.version>2.1.0-cdh6.3.2</hbase.version>
<solr.version>7.4.0-cdh6.3.2</solr.version>
<hive.version>2.1.1-cdh6.3.2</hive.version>
<kafka.version>2.2.1-cdh6.3.2</kafka.version>
<sqoop.version>1.4.7-cdh6.3.2</sqoop.version>
<zookeeper.version>3.4.5-cdh6.3.2</zookeeper.version>
```

在标签中新增如下依赖源，保存退出 !wq

```xml
<repository>
	<id>cloudera</id>
	<url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
	<releases>
		<enabled>true</enabled>
	</releases>
	<snapshots>
		<enabled>false</enabled>
	</snapshots>
</repository>  
```

为了兼容hive2.1.1版本，需要修改Atlas2.1.0默认Hive3.1的源代码，项目位置 /opt/apache-atlas-sources-2.1.0/addons/hive-bridge。
（1）修改文件./src/main/java/org/apache/atlas/hive/bridge/HiveMetaStoreBridge.java

```java
//第577行源代码：
String catalogName = hiveDB.getCatalogName() != null ? hiveDB.getCatalogName().toLowerCase() : null;
//修改为：
String catalogName = null;
```

（2）修改.src/main/java/org/apache/atlas/hive/hook/AtlasHiveHookContext.java

```java
//第81行源代码：
this.metastoreHandler = (listenerEvent != null) ? metastoreEvent.getIHMSHandler() : null;
//修改为：
this.metastoreHandler = null;
```

修改完年后，等待Maven依赖包全部下载完成。

打开命令窗口Terminal行进行编译。

```powershell
mvn clean  -DskipTests package -Pdist  -Drat.skip=true
```

等待编译安装。期间爆红提示，下面这句是问题核心。

```shell
Failure to find org.apache.lucene:lucene-core:jar:7.4.0-cdh6.3.2 in https://maven.aliyun.com/repository/public was cached in the local repository, resolution will not be reattempted until the update interval of aliyunmaven has elapsed or updates are forced
```

或

```shell
Could not find artifact org.apache.lucene:lucene-parent:pom:7.4.0-cdh6.3.2 in aliyunmaven (https://maven.aliyun.com/repository/public) ->[Help 1]
```

去maven仓库找到对应文件夹，如我这里本机路径D:\apache-maven-3.6.1\repository\org\apache\lucene\lucene-core\7.4.0-cdh6.3.2只保留里面的.jar和.pom文件，其他文件如.repositories、.jar.lastUpdated、.jar.sha1、.pom.lastUpdated、.pom.sha1全部删除，然后重新编译。如果还在此处报错，就去https://repository.cloudera.com/artifactory/cloudera-repos/中找到对应的缺失文件，放到本地仓库中，然后重编译。

编译完成后，在distro/target中可以看到apache-atlas-2.1.0-bin.tar.gz文件，将这个文件解压到CM server节点/data/software/atlas，并解压。

```
tar -zxvf apache-atlas-2.1.0-bin.tar.gz 
```

### 2.1 修改atlas配置文件

在atlas安装目录/conf目录下有 atlas-application.properties、atlas-log4j.xml、atlas-env.sh

```shell
cd  apache-atlas-2.1.0/conf
-------------------------------------
-rw-r--r-- 1 root root 12411 3月  24 15:00 atlas-application.properties
-rw-r--r-- 1 root root  3281 3月  24 15:13 atlas-env.sh
-rw-r--r-- 1 root root  5733 3月  24 15:03 atlas-log4j.xml
-rw-r--r-- 1 root root  2543 5月  25 2021 atlas-simple-authz-policy.json
-rw-r--r-- 1 root root 31403 5月  25 2021 cassandra.yml.template
drwxr-xr-x 2 root root    18 3月  24 15:15 hbase
drwxr-xr-x 3 root root   140 5月  25 2021 solr
-rw-r--r-- 1 root root   207 5月  25 2021 users-credentials.properties
drwxr-xr-x 2 root root    54 5月  25 2021 zookeeper
```

atlas-application.properties修改：

```properties
#修改hbase
atlas.graph.storage.hostname=hadoop01:2181,hadoop02:2181,hadoop03:2181
atlas.graph.storage.hbase.regions-per-server=1
atlas.graph.storage.lock.wait-time=10000
#修改solr
atlas.graph.index.search.solr.zookeeper-url=192.168.1.185:2181/solr,192.168.1.186:2181/solr,192.168.1.188:2181/solr

#等于false为外置的kafka
atlas.notification.embedded=false
atlas.kafka.zookeeper.connect=192.168.1.185:2181,192.168.1.186:2181,192.168.1.188:2181
atlas.kafka.bootstrap.servers=192.168.1.185:9092,192.168.1.186:9092,192.168.1.188:9092
atlas.kafka.zookeeper.session.timeout.ms=60000
atlas.kafka.zookeeper.connection.timeout.ms=60000


#修改其他配置
#默认访问端口21000，此端口和impala冲突，可以在cm中修改impala端口,因为已经安装了imapala,所以修改此处端口。
atlas.server.http.port=21021
atlas.rest.address=http://hadoop01:21021	
#如果设置为true，则在服务器启动时会运行安装步骤
atlas.server.run.setup.on.start=false		   

# hbase的zk集群节点
atlas.audit.hbase.zookeeper.quorum=hadoop01:2181,hadoop02:2181,hadoop03:2181
#添加hive
######### Hive Hook Configs #######
atlas.hook.hive.synchronous=false
atlas.hook.hive.numRetries=3
atlas.hook.hive.queueSize=10000
atlas.cluster.name=primary
```

atlas-log4j.xml修改：

```xml
去掉如下代码部分注释第79行-95行
<!-- Uncomment the following for perf logs -->

<!--
    <appender name="perf_appender" class="org.apache.log4j.DailyRollingFileAppender">
        <param name="file" value="${atlas.log.dir}/atlas_perf.log" />
        <param name="datePattern" value="'.'yyyy-MM-dd" />
        <param name="append" value="true" />
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d|%t|%m%n" />
        </layout>
    </appender>

    <logger name="org.apache.atlas.perf" additivity="false">
        <level value="debug" />
        <appender-ref ref="perf_appender" />
    </logger>
    -->
```

atlas-env.sh修改：新增 export HBASE_CONF_DIR=/etc/hbase/conf

```sh
#!/usr/bin/env bash
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# The java implementation to use. If JAVA_HOME is not found we expect java and jar to be in path
#export JAVA_HOME=

# any additional java opts you want to set. This will apply to both client and server operations
#export ATLAS_OPTS=

# any additional java opts that you want to set for client only
#export ATLAS_CLIENT_OPTS=

# java heap size we want to set for the client. Default is 1024MB
#export ATLAS_CLIENT_HEAP=

# any additional opts you want to set for atlas service.
#export ATLAS_SERVER_OPTS=

# indicative values for large number of metadata entities (equal or more than 10,000s)
#export ATLAS_SERVER_OPTS="-server -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+PrintTenuringDistribution -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=dumps/atlas_server.hprof -Xloggc:logs/gc-worker.log -verbose:gc -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1m -XX:+PrintGCDetails -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps"

# java heap size we want to set for the atlas server. Default is 1024MB
#export ATLAS_SERVER_HEAP=

# indicative values for large number of metadata entities (equal or more than 10,000s) for JDK 8
#export ATLAS_SERVER_HEAP="-Xms15360m -Xmx15360m -XX:MaxNewSize=5120m -XX:MetaspaceSize=100M -XX:MaxMetaspaceSize=512m"

# What is is considered as atlas home dir. Default is the base locaion of the installed software
#export ATLAS_HOME_DIR=

# Where log files are stored. Defatult is logs directory under the base install location
#export ATLAS_LOG_DIR=

# Where pid files are stored. Defatult is logs directory under the base install location
#export ATLAS_PID_DIR=

# where the atlas titan db data is stored. Defatult is logs/data directory under the base install location
#export ATLAS_DATA_DIR=

# Where do you want to expand the war file. By Default it is in /server/webapp dir under the base install dir.
#export ATLAS_EXPANDED_WEBAPP_DIR=

#hbse配置文件路径
export HBASE_CONF_DIR=/etc/hbase/conf

# indicates whether or not a local instance of HBase should be started for Atlas
#使用外部hbase，不用atlas内置hbase
export MANAGE_LOCAL_HBASE=false

# indicates whether or not a local instance of Solr should be started for Atlas
#使用外部solr，不使用atlas内置solr
export MANAGE_LOCAL_SOLR=false

# indicates whether or not cassandra is the embedded backend for Atlas
#使用外部cassandra，不使用atlas内置cassandra
export MANAGE_EMBEDDED_CASSANDRA=false

# indicates whether or not a local instance of Elasticsearch should be started for Atlas
#使用外部es,不使用atlas内置es
export MANAGE_LOCAL_ELASTICSEARCH=false
```

## 3 组件服务集成

### 3.1 集成CDH中HBase

将hbase配置文件添加到atlas的conf/hbase中。

```
ln -s /etc/hbase/conf  /data/software/atlas/apache-atlas-2.1.0/conf/hbase/
```

### 3.2 集成CDH中Solr

将atlas/conf/solr文件夹拷贝到所有安装solr节点的目录下，并更名为atlas-solr

```sh
cp -r atlas/conf/solr /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr
#所有solr节点执行
cd /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr
#所有solr节点执行
mv solr atlas-solr
#所有solr节点执行,修改solr用户对应的bash
vi /etc/passwd
/sbin/nologin 修改为 /bin/bash

useradd atlas && echo atlas | passwd --stdin atlas
chown -R atlas:atlas /usr/local/src/solr/

# solr节点创建collection
# 切换solr用户执行
su solr

/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/bin/solr create -c  vertex_index -d /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/atlas-solr -shards 3 -replicationFactor 1
/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/bin/solr create -c  edge_index -d /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/atlas-solr -shards 3 -replicationFactor 1
/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/bin/solr create -c  fulltext_index -d /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/solr/atlas-solr -shards 3 -replicationFactor 1
```

查看solr服务节点web页面，http://cdh001:8983，验证是否创建成功，出现如下内容。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6217c98b7eca4d72bd257f0c98a261d2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

### 3.3 集成CDH中Kafka

```sh
#创建测试Topic
kafka-topics --zookeeper cdh185:2181,cdh186:2181,cdh188.com:2181 --create --replication-factor 3 --partitions 3 --topic _HOATLASOK
kafka-topics --zookeeper cdh185:2181,cdh186:2181,cdh188.com:2181, --create --replication-factor 3 --partitions 3 --topic ATLAS_ENTITIES
kafka-topics --zookeeper cdh185:2181,cdh186:2181,cdh188.com:2181, --create --replication-factor 3 --partitions 3 --topic ATLAS_HOOK
#查看Topic列表
kafka-topics --zookeeper cdh185:2181 --list 
```

添加Atlas到系统变量

vim /etc/profile

```shell
#---------------- atlas ---------------------------
export ATLAS_HOME=/data/software/atlas/apache-atlas-2.1.0
export PATH=$PATH:$ATLAS_HOME/bin
```

启动Atlas

```shell
#启动命令
atlas_start.py
starting atlas on host localhost
starting atlas on port 21021
...................
Apache Atlas Server started!!!


# 查看端口启用状态
netstat -nultap | grep 21021
tcp        0      0 0.0.0.0:21021           0.0.0.0:*               LISTEN     
tcp        0      0 192.168.1.185:21021     172.16.10.11:51805      TIME_WAIT  
tcp        0      0 192.168.1.185:21021     172.16.10.11:51806      TIME_WAIT  
tcp        0      0 192.168.1.185:21021     172.16.10.11:51809      TIME_WAIT  
tcp        0      0 192.168.1.185:21021     172.16.10.11:51804      TIME_WAIT  
tcp        0      0 192.168.1.185:21021     172.16.10.11:51808      TIME_WAIT  
tcp        0      0 192.168.1.185:21021     172.16.10.11:51807      TIME_WAIT 

#页面查看，登录http://hadoop01:21021,默认账密admin/admin

#停止命令
atlas_stop.py
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/dd07f45f835c4b339019cde1db25a697.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

### 3.4 集成CDH中Hive

- 所有节点添加hive环境变量

```bash
vim /etc/profile 
#>>> 
#hive 
export HIVE_HOME=/opt/cloudera/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373/lib/hive 
export HIVE_CONF_DIR=/etc/hive/conf 
export PATH=$HIVE_HOME/bin:$PATH 
#<<< 

source /etc/profile
```

CM界面进行Hive配置文件hive-site.xml，

（1）修改【hive-site.xml的Hive服务高级配置代码段(安全阀)】

名称：hive.exec.post.hooks

值：org.apache.atlas.hive.hook.HiveHook

![在这里插入图片描述](https://img-blog.csdnimg.cn/d4137fcedba5459f9a629eeb12ae5fd2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

（2）修改【hive-site.xml的Hive客户端高级配置代码段(安全阀)】

名称：hive.exec.post.hooks

值：org.apache.atlas.hive.hook.HiveHook

![在这里插入图片描述](https://img-blog.csdnimg.cn/cbeec8b93f9b457aa549e914f2709192.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

（3）修改 【hive-site.xml 的 HiveServer2 高级配置代码段（安全阀）】

名称：hive.exec.post.hooks

值：org.apache.atlas.hive.hook.HiveHook

名称：hive.reloadable.aux.jars.path

值：HIVE_AUX_JARS_PATH=/data/software/atlas/apache-atlas-2.1.0/hook/hive

![在这里插入图片描述](https://img-blog.csdnimg.cn/2e20a27c90ca46cba6988c5f50375028.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

（4）修改 【HiveServer2 环境高级配置代码段（安全阀）】

HIVE_AUX_JARS_PATH=/data/software/atlas/apache-atlas-2.1.0/hook/hive

![在这里插入图片描述](https://img-blog.csdnimg.cn/0cbc5fd142a44ba594ddaf42605a18f8.png)

（5）atlas-application.properties 配置文件复制到/etc/hive/conf目录

```shell
cp /data/software/atlas/apache-atlas-2.1.0/conf/atlas-application.properties   /etc/hive/conf
```

（6）将atlas-application.properties 配置文件复制到atlas/hook/hive目录，压缩配置文件到atlas-plugin-classloader-2.1.0.jar中

```shell
#复制文件
cp /data/software/atlas/apache-atlas-2.1.0/conf/atlas-application.properties /data/software/atlas/apache-atlas-2.1.0/hook/hive
#进入目录
cd /data/software/atlas/apache-atlas-2.1.0/hook/hive
#配置文件压缩到atlas-plugin-classloader-2.1.0.jar
zip -u atlas-plugin-classloader-2.1.0.jar  atlas-application.properties
```

（7）Atlas导入hive元数据。

```shell
import-hive.sh 

#账密 admin/admin

#出现Hive Meta Data imported successfully!!! 成功导入hive元数据。
#通过atlas页面，查询可看到hive_db后面有数字即可。
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/dcc8aa1e3e6c4c11876e53ce503a1d16.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA55m-5aSc77mN5oKg44K8,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/fa053375d0c8487f8010c57c65f12dd6.png)

至此Atlas集成CDH集群完成。

## 附录A

### A.1 CDH6.3.2 集成 atlas2.1.0的各种问题

1.java.lang.AbstractMethodError: javax.ws.rs.core.UriBuilder.uri(Ljava/lang/String;)Ljavax/ws/rs/core/UriBuilder 的问题

![img](https://img-blog.csdnimg.cn/f76be7b25a274191bf2a70e0ffb7b22d.png)

解决办法删除掉javax.ws.rs-[api](https://so.csdn.net/so/search?q=api&spm=1001.2101.3001.7020)-2.1.jar

linux下搜素 `sudo find / -name javax.ws.rs-api*`，我的目录在/opt/module/atlas下面`/opt/module/atlas/server/webapp/atlas/WEB-INF/lib/javax.ws.rs-api-2.1.jar`，删除掉javax.ws.rs-api-2.1.jar

2.Class org.openx.data.jsonserde.JsonSerDe not found

下载 json-serde-1.3.8-jar-with-dependencies.jar和 json-udf-1.3.8-jar-with-dependencies.jar
放入 /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/hive/lib
记得分发到其他集群下面

### A.2 import_hive.sh报找不到方法的错

注意：CDH6.3.0安装的HIVE版本为2.1.0， 但Atlas 2.1.0版本中编译用的HIVE版本为3.x的。

在执行import_hive.sh时会报找不到方法的错。需要修改下源代码里的方法，重新编译包hive-bridge-2.1.0.jar所需修改的项目位置：`apache-atlas-sources-2.1.0\addons\hive-bridge`org/apache/atlas/hive/bridge/HiveMetaStoreBridge.java 577行。

```java
public static String getDatabaseName(Database hiveDB){
    String dbName = hiveDB.getName.toLowerCase();
    
    //注释掉下面这部分
    /** 
    String catalogName = hiveDB.getCatalogName()!=null ? hiveDB.getCatalogName().toLowerCase:null;
    if(StringUtils.isNotEmpty(catalogName)&& !StringUtils.equlas(catalogName,DEFAULT_METASTORE_CATALOG)){
        dbName=catalogName+SEP+dbName;
    }**/
    return dbName;
}
```

修改后重新编译,将apache-atlas-sources-2.1.0/addons/hive-bridge/target/hive-bridge-2.1.0.jar，替换${ATLAS_HOME}/hook/hive/atlas-hive-plugin-impl/hive-bridge-2.1.0.jar

### A.3 lucene-core-7.4.0-cdh6.3.2.jar 无法下载。

中央库Cloudera-repl中存在，下载显示404

```delphi
[ERROR] Failed to execute goal on project atlas-graphdb-janus: Could not resolve dependencies for project org.apache.atlas:atlas-graphdb-janus:jar:2.1.0: Failed to collect dependencies at org.apache.lucene:lucene-core:jar:7.4.0-cdh6.3.2: Failed to read artifact descriptor for org.apache.lucene:lucene-core:jar:7.4.0-cdh6.3.2: Could not transfer artifact org.apache.lucene:lucene-solr-grandparent:pom:7.4.0-cdh6.3.2 from/to cloudera (https://repository.cloudera.com/artifactory/cloudera-repos/): Tra
```

solution：获取lucene-core-7.4.0.jar换名字lucene-core-7.4.0-cdh6.3.2.jar，比较无耻但有用。