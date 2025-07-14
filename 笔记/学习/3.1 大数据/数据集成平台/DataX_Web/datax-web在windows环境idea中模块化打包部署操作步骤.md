- [datax-web在windows环境idea中模块化打包部署操作步骤_GIS_hot的博客-CSDN博客_datax-web windows部署](https://blog.csdn.net/u012874077/article/details/116148000)

datax-web在windows环境idea中模块化打包部署

操作步骤：

1.在idea中点击[Maven](https://so.csdn.net/so/search?q=Maven&spm=1001.2101.3001.7020) Projects，点击datax-web中的clean，等待执行完成后，点击package进行打包。等待打包完成。

![img](https://img-blog.csdnimg.cn/20210426112849140.png)



2.打包完成后，拷贝datax-admin下的target/datax-admin-2.1.2.jar,以及datax-executor下的target/datax-executor-2.1.2.jar

![img](https://img-blog.csdnimg.cn/20210426113153904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210426113236783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

3.打开cmd，执行java -jar C:\Users\Administrator\Desktop\datax-web\datax-admin-2.1.2.jar，发现报错，提示没有主目录清单

![img](https://img-blog.csdnimg.cn/20210426142609887.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

4.若报错，提示没有主目录清单，那么分别在datax-admin和datax-executor下面的pom.xml文件中添加

```html
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
    <excludes>
        <exclude>**/*.yml</exclude>
        <exclude>**/*.properties</exclude>
        <exclude>**/*.sh</exclude>
        <exclude>**/*.xml</exclude>
    </excludes>
    </configuration>
</plugin>
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

5.添加完后，重新步骤1、2、3。执行打包生成后的datax-admin-2.1.2.jar和datax-executor-2.1.2.jar,结果还是报错，出现报错结果如下：

datax-admin-2.1.2.jar报错如下：

![img](https://img-blog.csdnimg.cn/20210426143505259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

datax-executor-2.1.2.jar报错如下：

![img](https://img-blog.csdnimg.cn/20210426143634175.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

6.发现问题：打包后的datax-admin.2.1.2.jar和datax-executor.2.1.2.jar包缺少配置文件，应把配置文件补充进jar包

7.将datax-admin下的所有配置资源拷贝进datax-admin-2.1.2.jar的相关目录中。具体 如下：

 1）将idea中的datax-admin下的classes下的配置文件application.yml、bootstrap.properties、logback.xml拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes下。

![img](https://img-blog.csdnimg.cn/20210426152051177.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210426143921475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

​     2）将datax-admin下的target/classes/i8n下的message.properties和message_en.properties拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes\i18n下

![img](https://img-blog.csdnimg.cn/20210426151828702.png)![img](https://img-blog.csdnimg.cn/20210426152003413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

​     3）将将datax-admin下的target/classes/mybatis-mapper下的所有xml文件拷贝进datax-admin-2.1.2.jar下的BOOT-INF\classes\mybatis-mapper下

![img](https://img-blog.csdnimg.cn/20210426152813548.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210426152849498.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

8.cmd执行命令，启动datax-admin,启动成功

![img](https://img-blog.csdnimg.cn/20210426153141499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210426153240174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

9.将datax-executor下的所有配置资源拷贝进datax-executor-2.1.2.jar的相关目录中。具体 如下：

1)将idea中的datax-executor编译生成后的classes下的配置文件application.yml、logback.xml拷贝进datax-executor-2.1.2.jar下的BOOT-INF\classes下。

![img](https://img-blog.csdnimg.cn/20210426153547839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210426153650830.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

10.cmd执行命令，启动datax-executor,启动成功

![img](https://img-blog.csdnimg.cn/20210426153903746.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20210426153940797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

11.访问http://127.0.0.1:8080/index.html,账号输入admin,密码输入123456,登录成功！

![img](https://img-blog.csdnimg.cn/2021042615420752.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20210426154254319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

12.点击数据源管理，新建数据源配置，在点击测试数据源时，报错Failed to load driver class oracle.jdbc.OracleDriver in either of HikariConfig class loader or Thread context classloader，

无法加载oracle驱动。如下图所示

![img](https://img-blog.csdnimg.cn/2021050716583671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

13.发现问题后，发现打包的时候缺少了3个驱动包，如下图所示：

![img](https://img-blog.csdnimg.cn/20210507164257494.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

14.我们把缺少的3个jar包，补充进datax-admin-2.1.2.jar下的BOOT-INF\lib下。

注意：补充进的时候需要**设置不压缩直接存储，否则会报错，不识别压缩的驱动jar包**

![img](https://img-blog.csdnimg.cn/20210507164652833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20210507165249607.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20210507165326846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)

15.新建数据源管理问题解决

![img](https://img-blog.csdnimg.cn/20210507165711421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI4NzQwNzc=,size_16,color_FFFFFF,t_70)