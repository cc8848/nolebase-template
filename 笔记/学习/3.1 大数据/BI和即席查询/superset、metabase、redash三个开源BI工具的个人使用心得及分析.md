- [superset、metabase、redash三个开源BI工具的个人使用心得及分析](https://blog.csdn.net/weixin_42473019/article/details/105419781)

[数据可视化](https://so.csdn.net/so/search?q=数据可视化&spm=1001.2101.3001.7020)也是大数据领域里极为关键的一环，通过计算引擎算出来的数据往往需要以合适又美观的图表形式展示给产品经理和决策者，一开始笔者的部门用的是SpringBoot+ECharts的经典组合来做可视化的，确实功能很强也很灵活，但后来随着统计需求越来越多，每来一个新需求都要写大量的重复的MVC代码，这等于花费了许多无用的时间和精力，这种时候就需要一个BI工具来简化这个过程。
市面上有很多成熟优秀的BI，像[Tableau](https://so.csdn.net/so/search?q=Tableau&spm=1001.2101.3001.7020)等，但这些都是收费的，不在我们考虑范围内。而开源免费的BI目前比较流行的有superset、metabase、redash这三款，功能都比较强大，各有各的特点。
这三款BI我都有调研并安装使用过，最终根据自身的业务需求，选择了metabase作为生产主力，redash作为辅助，没有使用superset。
下表是我个人实际体验后对这三款BI一些核心功能的简单总结：

| 项目       | metabase | redash | superset |
| ---------- | -------- | ------ | -------- |
| 安装难度   | 低       | 高     | 中       |
| 上手难度   | 中       | 低     | 高       |
| 数据源类型 | 少       | 多     | 多       |
| 图表类型   | 中       | 少     | 多       |
| 图表颜值   | 高       | 中     | 低       |
| 文档教程   | 强       | 中     | 弱       |
| 项目活跃度 | 高       | 中     | 低       |
| 邮件功能   | 强       | 中     | 无       |

# 一、安装难度

## metabase

metabase的安装及其简单，只需要去[官网](https://www.metabase.com/)下载一个jar包到服务器，然后**java -jar metabase.jar**就可以启动了，默认数据库为h2，端口为3000，如果想修改，可以参考一下这篇文章：https://www.mekau.com/6136.html

```powershell
# h2(默认)
export MB_DB_TYPE=h2
export MB_DB_FILE=/the/path/to/my/h2.db
java -jar metabase.jar
#mysql
export MB_DB_TYPE=mysql
export MB_DB_DBNAME=metabase
export MB_DB_PORT=3306
export MB_DB_USER=<username>
export MB_DB_PASS=<password>
export MB_DB_HOST=localhost
java -jar metabase.jar
# 端口
export MB_JETTY_PORT=12345
export MB_JETTY_HOST=0.0.0.0
export MB_JETTY_SSL="true"
export MB_JETTY_SSL_Port="8443"
export MB_JETTY_SSL_Keystore="path/to/keystore.jks"
export MB_JETTY_SSL_Keystore_Password="storepass"
 
java -jar -Xms2048M -Xmx2048M -XX:NewSize=512m -XX:MaxNewSize=1024m -XX:PermSize=512m -XX:MaxPermSize=512m metabase.jar

```

## redash

centos上安装redash是最麻烦的，当初花了不少时间，我也写了一篇文章专门介绍如何在centos7安装redash，需要的可以移步[超详细的centos7安装redash教程](https://blog.csdn.net/weixin_42473019/article/details/104155959)

## superset

superset的安装难度一般，跟着官网的步骤一步一步来就行了，[官网地址](http://superset.apache.org/installation.html#getting-started)

# 二、上手难度

## metabase

metabase的上手难度我认为一般，只是经常有一些小坑要花时间去摸索，官网也有非常详细的教程文档

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410163316734.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

上图为metabase新建一个查询的三种方式，

第一种功能太过简单一般不会用；

第二种跟superset很像，不需要写sql，只需要选择一张表，选择筛选条件、聚合维度、计算函数、排序方式这些东西，就能得到一个展示页面，这种方式我认为主要面向不熟悉sql的产品经理或是领导；

第三种就是写一条sql，选择一种图表类型将这条sql查询出的数据展示出来，作为一个技术人员，这种方式也是我最喜欢最常用的方式，具有很强的灵活性，支持多表join，其中的变量功能也很强大，官方还贴心的在旁边放上了变量功能的帮助文档

## redash

redash上手难度简单，它只有一种查询方式，就是上面的第三种写sql的方式，它的一些功能比metabase更为强大，比如它支持在group by语句里嵌入变量，这样的话只需要选择不同变量参数，就能在一张表看到不同维度的数据。

但是它的变量功能也有一个坑点，好像是不支持为空的，在metabase里 **[[WHERE category = {{category}}]]** 这样的写法可以在category变量为空的时候忽略整条语句，这个功能可以说非常重要，但是我不知道redash是不支持这样的功能还是我没有找到应该怎样实现，这里也体现出metabase在教程文档方面的优势，它对于变量功能的使用有非常完善的帮助文档

## superset

superset上手可以说是比较难的，它最主要的新建查询方式跟metabase第二种方式很像，同时它也有一个SQL Lab功能，superset我使用的不多，它在各种像是配置数据源、建立查询的时候都会比上面两者更加复杂，而且它的文档方面也是最弱的，遇到问题也不好找教程

# 三、数据源类型

## metabase

metabase的数据源种类是最少的，自带的可选数据源如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020041017085827.png)

值得一提的是metabase为我们提供了二次开发添加数据源的方式，这部分官网也有详细介绍：https://www.metabase.com/docs/latest/developers-guide-drivers.html

## redash

这是redash支持的数据源：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410170944376.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

## superset

这是superset支持的数据源：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410171015866.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

# 四、图表类型

## metabase

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410171233258.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

## redash

官网有样例展示：https://redash.io/help/user-guide/visualizations/visualization-types

实际操作里可选的图表如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410171303350.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410171310786.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

## superset

superset支持的图表类型是最多的，不愧是老牌劲旅，眼睛都看花了：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410171502129.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

# 五、图表颜值

颜值这方面纯属个人主观，毕竟每个人审美不同，在我看来metabase的颜值最高，redash其次，而superset因为资历太老，审美有点跟不上时代，下面依次是metabase、redash、superset的图表展示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412113608168.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412113619225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412113630776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

# 六、文档教程

这一块上面也提到了，metabase最强，redash次之，superset最弱

# 七、项目活跃度

活跃度决定了一个项目后续的发展和BUG的优化，在这方面metabase同样是最强的，在我使用过程中就有好几次更新，redash次之，而superset虽然目前属于apache，在github上星数也是最多的，但活跃度却是最弱的，发展前景看起来也一般

# 八、邮件功能

邮件功能在实际工作里是非常重要的一项功能，很多时候部门领导和产品需要每天收到一份关键数据的日报邮件，同样对于数据的异常情况也可以发送邮件进行告警，在这方面metabase还是最强的，首先它有一个定时任务功能，如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410175702868.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

在这里可以选择需要发送的图表，设置发送时间和接收人，就能实现日报功能

**Tips:这里有一个小坑，metabase默认时区是美国的时区，要在管理员页面设置成香港时区**

同时metabase每个图表都有一个告警功能

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410180138928.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3MzAxOQ==,size_16,color_FFFFFF,t_70)

非常好用，美中不足的一点是最小的告警时间间隔只能到一个小时，某些情况下可能粒度不够细，redash也有告警功能，时间间隔可以到一分钟，大部分情况下通过告警功能也可以实现日报的效果
而superset是没有邮件功能的

# 九、总结

从上面各方面的对比可以看出，为什么我把metabase作为了生产主力，同时还要用redash来辅助

## metabase

metabase各方面的表现已经基本能满足工作要求，但仍有几个缺陷：

- 第一个就是支持的数据源比较少，我的数据源主要有Mysql和ClickHouse两种，metabase默认是不支持ClickHouse的，所以我需要用redash来展示ClickHouse的数据；
- 第二个缺点是metabase一些图表的功能不全，比如在多根线条的折线图里，它最多只能支持20条线(目前新版本中应该修复了这个问题)，有些时候这是不够的

抛开这两点的话metabase有极为优秀的表现，无论是在图表展示方面还是日报邮件方面都对我的工作起到了很大帮助

## redash

上面metabase的两个问题redash都能完美解决，但它也有一些问题：

1. 在redash里制作出的图表不会随着页面的刷新而刷新，用户需要在图表里配置一个**Refresh Schedule**：刷新间隔，最小间隔是一分钟，这样redash会在后台每一分钟刷新一次这个图表，而在后台不刷新的时候，无论怎么刷新页面看到的都是一分钟前的数据，仔细想想这个特点其实有好处也有坏处，首先坏处是数据不够及时，而好处是对于那种需要长时间计算的图表来说用户体验会很好，每次打开都能马上看到数据，无需等待计算的过程
2. 还有一个缺陷是上面提到过的redash的变量参数不能为空，所以redash在一些特定情况下会很适合，可以作为一个辅助使用

## superset

而superset因为操作复杂、颜值逊色、没有邮件功能等原因，我并没有在生产中使用它，但这并不意味着superset是一款不好的BI，从它隶属于apache这一点就可以看出它有着深厚的底蕴，只是在挑选一款工具的时候我们要结合自身情况，合适才是最重要的