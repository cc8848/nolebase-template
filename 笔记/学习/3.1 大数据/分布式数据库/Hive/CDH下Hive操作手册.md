- [CDH之HIVE-ON-SPARK、Spark配置](https://blog.csdn.net/Yellow_python/article/details/124921416)

## 1 Hive初始化MySQL

- [CDH hive初始化mysql数据库](https://blog.csdn.net/c1z2w3456789/article/details/123199240)

进入hive的bin目录下：`cd ./hive/bin`

执行初始化数据库

```bash
./schematool -initSchema -dbType mysql
# 如果报错，提示Schema script failed, errorcode 2，则查看hive默认数据库，执行如下命令
./schematool -initSchema -dbType default
```

## 2 Hive修改derby为mysql数据库

- [Hive修改derby为mysql数据库](https://blog.csdn.net/yu342107056/article/details/88859871)

### 2.1 概述

1. hive可以管理hdfs上的文件，用表的形式来管理文件数据。而表名、表里有哪些字段，字段类型、哪张表存在哪个数据下等这些表信息，称之为hive的元数据信息
2. 默认情况下，hive的元数据信息不是存在hdfs上的，而是存在hive自带的derby关系型数据库里的
3. Hive安装完成之后，通常都需要替换元数据库，目前Hive只支持derby和mysql两种元数据库，需要将元数据库替换为MySQL
4. 元数据的默认字符集是ISO8859-1

### 2.2 derby存在的问题

1. derby数据库是一种文件型的数据库，在进入时会检查当前目录下是否有metastore_db文件夹用来存储数据库数据，如果有就直接使用，如果没有就创建，这样一旦换一个目录，元数据就找不到了
2. derby数据库是一个单用户的数据库，无法支持多用户同时操作，而hive如果使用derby作为元数据库，则也只能支持单用户操作，这就导致在数据量大连接多的情况下会产生大量连接的积压

### 2.3 替换为MySQL

所以每次在不同的目录启动hive都会生成不同的derby数据文件，所以很麻烦，不如直接替换成MySQL使用
不同的目录指的是在当前目录通过路径的方式启动hive 例如在home目录下 sh /home/software/hive-1.2.2/bin/hive,就会在home目录下生成metastore_db文件

1. 将mysql驱动包上传到hive安装目录的lib目录下

下载地址：http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.38/

```xml
cp mysql-connector-java-5.1.38.jar /home/software/hive-1.2.2/lib/
```

2. 进入conf目录下，编辑新的配置文件，名字为：hive-site.xml

3. 配置相关信息：

数据库的连接信息，按照自己的数据库填写既可

```xml
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop01:3306/hive?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name> 
        <value>root</value> 
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>root</value>
    </property>
</configuration>
```

4. 进入hive ,进入bin目录，执行：`sh hive`

如果出现：Access denied for user ‘root’@‘hadoop01’ (using password: YES）这个错误，指的是当前用户操作mysql数据库的权限不够

5. 使用连接工具到mysql数据库，进行权限分配

```sql
grant all privileges on *.* to 'root'@'hadoop01' identified by 'root' with grant option;
grant all on *.* to 'root'@'%' identified by 'root';
flush privileges;
```

6. 进入mysql数据库，执行：`create database hive character set latin1;`

7. 启动hive客户端：`sh hive`

8. 以上步骤都做完后，再次进入mysql的hive数据，发现有如下的表：

9. 可以通过DBS 、TBLS、COLUMNS_V2、SDS这几张表来查看元数据信息，用hive创建一个表：

```sql
create table stu(id int,name varchar(50));
```

- DBS 存放的数据库的元数据信息

- TBLS存放的tables表信息

-  COLUMNS表存放的是列字段信息

- SDS表存放的HDFS里的位置信息

## 3 CDH环境下的Hive数仓配置及优化

- [基于CDH环境下的Hive数仓配置及优化](https://blog.csdn.net/sinat_31854967/article/details/127274575)

# 4 CDH集群修改/设置hive的动态分区（dynamic）相关参数

- [CDH集群修改/设置hive的动态分区（dynamic）相关参数](https://blog.csdn.net/summer089089/article/details/106790593)

