- [HDFS权限管理设置_寒烟雾柳的博客-CSDN博客_dfs.permissions.enabled](https://blog.csdn.net/c18092694487t/article/details/116047485)

# 一、HDFS权限相关的配置参数

## 1.1.配置HDFS权限

可以通过在"${HADOOP_HOME}/etc/hadoop/hdfs-site.xml"配置文件中将参数"dfs.permissions.enabled"的值设置为ture来配置HDFS权限，也可在cm中设置dfs.permissions.enabled 为ture。
　　如下所示，有此参数的默认值为true，因此权限检查功能已经打开，故HDFS不需要再执行其他任何操作来进行权限检查。

```xml
<property>
 　　<name>dfs.permissions.enabled</name>
 　　<value>true</value>
 　　<description>如果为"true"，则在HDFS中启用权限检查;如果为"false"，则关闭权限检查;默认值为"true"。</description>			        
</property>
```

## 1.2.配置HDFS超级用户

与Linux文件系统不同，主机用户root不是HDFS的超级用户。超级用户是启动NameNode的用户，通常是操作系统用户(如"root",“hdfs”,“hadoop"等)，因此用户(如"root”,“hdfs”,"hadoop"等)通常是HDFS的超级用户。

可以通过在"${HADOOP_HOME}/etc/hadoop/hdfs-site.xml"配置文件中设置"dfs.permissions.superusergroup"参数来配置超级用户组(改参数仅能被设置一次)。

```xml
<property>
　　<name>dfs.permissions.superusergroup</name>
　　<value>root</value>
　　<description>配置超级用户组</description>
</property>
```

**注**：此参数指定包含HDFS超级用户的组(可以任意指定，分配给该组的任何用户都将是HDFS超级用户)，默认值为"supergroup"，我这里自定义超级用户的组名为"admin"。需要注意的是，该组名只能被设置一次，当对NameNode节点进行格式化后在修改改参数并不生效!如果你想要的强行生效只能重新格式化NameNode节点。幸运的是，我们可以通过"hdfs dfs -chown"命令来修改某个文件或目录的所属者。

# 二、更改HDFS集群中文件属性的常用命令

## 2.1.[chmod](https://so.csdn.net/so/search?q=chmod&spm=1001.2101.3001.7020)命令

chmod命令用于更改HDFS集群中文件的权限。这类似于shell的chmod命令，但有一些例外(使用"man chmod"，你会发现Linux系统支持的参数更多)。

```powershell
hdfs dfs -help chmod
hdfs dfs -chmod 700 /yinzhengjie/hosts
hdfs dfs -chmod -R 731 /yinzhengjie/conf
```

## 2.2. chown命令

更改文件的所有者和组。这与Linux shell的chown命令类似，但有一些例外。

```powershell
hdfs dfs -help chown
hdfs dfs -chown jason /yinzhengjie/hosts
hdfs dfs -chown :admin /yinzhengjie/hosts
hdfs dfs -chown java:bigdata /yinzhengjie/hosts
hdfs dfs -chown -R python:devops /yinzhengjie/conf
```

## 2.3.chgrp命令

chgrp命令相当于 “-chown …:GROUP …”

```powershell
hdfs dfs -help chgrp
hdfs dfs -chgrp supergroup /yinzhengjie/hosts
hdfs dfs -chgrp -R admingroup /yinzhengjie/conf
```

# 三、HDFS访问控制列表

HDFS支持使用ACL为特定用户和组设置更细粒度的权限。当希望以细粒度的方式授予权限，并处理复杂文件权限和访问需求时，ACL是一种很好的方式。

在默认情况下，ACL是被禁用的。为了使用HDFS ACL，可以通过将以下属性添加到"${HADOOP_HOME}/etc/hadoop/hdfs-site.xml"配置文件中，从而在NameNode上启用它。

```xml
<property>
　　<name>dfs.namenode.acls.enabled</name>
　　<value>true</value>
　　<description>设置为true以启用对HDFS acl（访问控制列表）的支持。默认情况下，ACL处于禁用状态。禁用ACL时，NameNode会拒绝与设置或获取ACL相关的所有RPC。</description>
</property>
```

**温馨提示**:若将"dfs.namenode.acls.enabled"修改为"true"，执行setfacl命令不会生效，出现如下图所示错误。别忘记重启HDFS集群哟~否则NameNode并不会让ACL生效，换句话说就是无法设置配置ACL策略。
![在这里插入图片描述](https://img-blog.csdnimg.cn/202104231347158.png#pic_center)

## 3.1.与ACL相关的命令概述

getfacl:显示文件和目录的访问控制列表（ACL）。如果目录具有默认ACL，那么getfacl也会显示默认ACL。
setfacl:设置文件和目录的访问控制列表（ACL）。

```powershell
hdfs dfs -help getfacl
hdfs dfs -help setfacl
```

## 3.2.使用getfacl命令检查目录或文件上当前的ACL信息(若之前没有启用ACL功能，默认是没有ACL策略的)

hdfs dfs -getfacl /yinzhengjie/hosts

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423134731354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 3.3.使用setfacl命令为目录或文件设置ACL策略

如下图所示，在为文件设置ACL策略后，在该文件权限列表的最后添加了一个加号("+")，一次表示此文件权限的ACL条目。这一点你是否似曾相识？基本上和Linux的ACL策略有异曲同工之妙。

需要注意的是，即使可以对具有ACL的文件或目录授予用户特定权限，但这些权限必须在文件掩码的范围之内。

如下图所示，在"-getfacl"命令的输出中，有一个名为mask的新条目，这里，它的值为"rwx"，这意味着用户(root)和组(root)在此文件上确实具有"rwx"权限。

mask属性定义对文件的最大权限。如果掩码是"r-x"，则虽然用户(jason)具有"rw-“权限，但有效的权限只是"r–”。

```powershell
hdfs dfs -setfacl -m user:root:rw- /yinzhengjie/hosts
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423134850866.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 3.4.使用chmod命令更改文件权限，而后在看它对新ACL掩码的影响

```powershell
 hdfs dfs -chmod 750 /yinzhengjie/hosts
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021042313494636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 3.5.删除ACL策略

```powershell
hdfs dfs -setfacl -b /yinzhengjie/hosts
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423135002576.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

# 四、HDFS扩展属性

HDFS允许用户往文件或者目录中添加一些其他的元数据，这些数据被称为HDFS的扩展属性，因此应用程序也可以将其他信息存储在inode中。例如，HDFS扩展属性可以帮助应用程序指定文档的字符编码。HDFS扩展属性可以被看作是系统HDFS文件系统权限的扩展。

如下图所示，在设置扩展属性(XAttr)时需要制定命名空间，否则就会设置属性失败。虽然有5种不同访问限制的命名空间类型(即"user",“trusted”,“security”,“system”,“raw”):

- 客户端应用程序仅使用"用户(user)"命名空间，"用户(user)"命名空间中的HDFS扩展属性可通过HDFS文件权限进行管理;
- 还有3种其他类型的命名空间，即"系统(system)"，"安全性(security)“和"原始(raw)”，供内部HDFS和其他系统使用;
- "可信(trusted)"命名空间是为HDFS超级用户保留的;

默认HDFS扩展属性为启用状态，可以通过"${HADOOP_HOME}/etc/hadoop/hdfs-site.xml"配置文件中的"dfs.namenode.xattrs.enabled"的值设置为"true"或"false"来启用或禁用该功能。

```xml
<property>
　　　<name>dfs.namenode.xattrs.enabled</name>
　　　<value>true</value>
　　　<description>是否在NameNode上启用了对扩展属性的支持。默认值为true</description>
</property>
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423135422241.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 4.1.查看文件或目录的扩展属性的名称和值

```powershell
hdfs dfs -help getfattr
hdfs dfs -getfattr -R -d /yinzhengjie/conf
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423135525892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 4.2.将扩展属性的名称和值与文件或目录相关联

```powershell
hdfs dfs -help setfattr
hdfs dfs -setfattr -n user.department -v DevOps /yinzhengjie/hosts
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423135541382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

## 4.3.删除扩展属性

```powershell
hdfs dfs -setfattr -x user.bigdata /yinzhengjie/hosts
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210423135601492.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2MxODA5MjY5NDQ4N3Q=,size_16,color_FFFFFF,t_70#pic_center)

参考：https://www.cnblogs.com/yinzhengjie2020/p/13737254.html