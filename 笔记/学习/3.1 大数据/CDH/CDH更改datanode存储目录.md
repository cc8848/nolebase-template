- [CDH更改datanode存储目录 - DavidZuo - 博客园 (cnblogs.com)](https://www.cnblogs.com/chong-zuo3322/p/16545828.html)

### 1.问题梳理：

CDH集群中，数据文件位置默认在 /[dfs](https://so.csdn.net/so/search?q=dfs&spm=1001.2101.3001.7020)/dn 中，这个目录所属的盘符是 /，但是 ‘/’只有50G大小，还有其他各种文件的日志什么的，空间很紧张。今天启动脚本上传文件到hive，因为空间问题一直报错。查阅了资料，需要将CDH集群 HDFS数据存储 更换目录。

#### 通过命令查看各个盘符 占用情况

```
df -h
```

![img](https://img2022.cnblogs.com/blog/1853022/202208/1853022-20220803091215712-1960534294.png)

 

 我当时占用100%。必须更换数据目录。

### 2. 处理

#### 1. 定位思路

先将集群停止，然后准备好新的存储目录，再将数据复制到新的目录中，重启集群。

#### 2.处理步骤

假设[HDFS](https://so.csdn.net/so/search?q=HDFS&spm=1001.2101.3001.7020)的默认安装目录为“/dfs/dn”，需要移动目录到“/home/dfs_new/dn”中。

1 .登录cdh manager，停止hdfs集群

2 .在home目录下创建 dfs_new/dn 目录

```
cd /home
mkdir -p   /home/dfs_new/dn
```

3.更改目录所属的用户和组

```
chown -R hdfs  /home/dfs_new/dn

chgrp -R hadoop /home/dfs_new/dn
```

使用cdh 搭建hdfs集群的时候，默认用户为hdfs ，默认用户组为hadoop

将数据复制到目标目录：当前DataNode的目录为根目录下，和系统目录在同一目录下，随着DataNode的增长导致系统运行空间不足

```
cp -af /dfs/dn/* /home/dfs_new/dn
```

4 . 在cdh Manager管理界面，更改DataNode的配置目录

打开HDFS服务datanode配置页面，将所有datanode的数据目录配置项“dfs.datanode.data.dir”

由“/dfs/dn”修改为“/home/dfs_new/dn”。

![img](https://img2022.cnblogs.com/blog/1853022/202208/1853022-20220803091346462-1990289801.png)

 

 5 . 启动成功后，执行fsck检查是否复制正确。 命令样例：

```
hdfs fsck /
```

6 . fsck显示有文件丢失，则检查2是否正确，并执行相关的修复。

7 . fsck显示文件无文件丢失，HDFS没有处于安全模式，则表示数据复制成功。

8 . 删除原有数据目录“/dfs/dn”中的文件。

9 . 启动集群中剩余的服务。

10 . 再平衡

![img](https://img2022.cnblogs.com/blog/1853022/202208/1853022-20220803091413307-1396554988.png)

 