- [分布式任务调度平台XXL-JOB (xuxueli.com)](https://www.xuxueli.com/xxl-job/#《分布式任务调度平台XXL-JOB》)

# 1 xxl-job概述

XXL-JOB是一个分布式任务调度平台，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。

## 1.1 xxl-job架构

![img](https://img-blog.csdnimg.cn/img_convert/92ad439e536c780213afef22f60f2c37.png)
我把上面的图精简了一下，xxl-job 的调度器和业务执行是独立的。调度器决定任务的调度，并且通过 http 的方式调用执行器接口执行任务。

![img](https://img-blog.csdnimg.cn/img_convert/3d66e0522d595fef84082601c33a4635.png)

## 1.2 特性

1、简单：支持通过Web页面对任务进行CRUD操作，操作简单，一分钟上手；

2、动态：支持动态修改任务状态、启动/停止任务，以及终止运行中任务，即时生效；

3、调度中心HA（中心式）：调度采用中心式设计，“调度中心”自研调度组件并支持集群部署，可保证调度中心HA；

4、执行器HA（分布式）：任务分布式执行，任务”执行器”支持集群部署，可保证任务执行HA；

5、注册中心: 执行器会周期性自动注册任务, 调度中心将会自动发现注册的任务并触发执行。同时，也支持手动录入执行器地址；

6、弹性扩容缩容：一旦有新执行器机器上线或者下线，下次调度时将会重新分配任务；

7、触发策略：提供丰富的任务触发策略，包括：Cron触发、固定间隔触发、固定延时触发、API（事件）触发、人工触发、父子任务触发；

8、调度过期策略：调度中心错过调度时间的补偿处理策略，包括：忽略、立即补偿触发一次等；

9、阻塞处理策略：调度过于密集执行器来不及处理时的处理策略，策略包括：单机串行（默认）、丢弃后续调度、覆盖之前调度；

10、任务超时控制：支持自定义任务超时时间，任务运行超时将会主动中断任务；

11、任务失败重试：支持自定义任务失败重试次数，当任务失败时将会按照预设的失败重试次数主动进行重试；其中分片任务支持分片粒度的失败重试；

12、任务失败告警；默认提供邮件方式失败告警，同时预留扩展接口，可方便的扩展短信、钉钉等告警方式；

13、路由策略：执行器集群部署时提供丰富的路由策略，包括：第一个、最后一个、轮询、随机、一致性HASH、最不经常使用、最近最久未使用、故障转移、忙碌转移等；

14、分片广播任务：执行器集群部署时，任务路由策略选择”分片广播”情况下，一次任务调度将会广播触发集群中所有执行器执行一次任务，可根据分片参数开发分片任务；

15、动态分片：分片广播任务以执行器为维度进行分片，支持动态扩容执行器集群从而动态增加分片数量，协同进行业务处理；在进行大数据量业务操作时可显著提升任务处理能力和速度。

16、故障转移：任务路由策略选择”故障转移”情况下，如果执行器集群中某一台机器故障，将会自动Failover切换到一台正常的执行器发送调度请求。

17、任务进度监控：支持实时监控任务进度；

18、Rolling实时日志：支持在线查看调度结果，并且支持以Rolling方式实时查看执行器输出的完整的执行日志；

19、GLUE：提供Web IDE，支持在线开发任务逻辑代码，动态发布，实时编译生效，省略部署上线的过程。支持30个版本的历史版本回溯。

20、脚本任务：支持以GLUE模式开发和运行脚本任务，包括Shell、Python、NodeJS、PHP、PowerShell等类型脚本;

21、命令行任务：原生提供通用命令行任务Handler（Bean任务，”CommandJobHandler”）；业务方只需要提供命令行即可；

22、任务依赖：支持配置子任务依赖，当父任务执行结束且执行成功后将会主动触发一次子任务的执行, 多个子任务用逗号分隔；

23、一致性：“调度中心”通过DB锁保证集群分布式调度的一致性, 一次任务调度只会触发一次执行；

24、自定义任务参数：支持在线配置调度任务入参，即时生效；

25、调度线程池：调度系统多线程触发调度运行，确保调度精确执行，不被堵塞；

26、数据加密：调度中心和执行器之间的通讯进行数据加密，提升调度信息安全性；

27、邮件报警：任务失败时支持邮件报警，支持配置多邮件地址群发报警邮件；

28、推送maven中央仓库: 将会把最新稳定版推送到maven中央仓库, 方便用户接入和使用;

29、运行报表：支持实时查看运行数据，如任务数量、调度次数、执行器数量等；以及调度报表，如调度日期分布图，调度成功分布图等；

30、全异步：任务调度流程全异步化设计实现，如异步调度、异步运行、异步回调等，有效对密集调度进行流量削峰，理论上支持任意时长任务的运行；

31、跨语言：调度中心与执行器提供语言无关的 RESTful API 服务，第三方任意语言可据此对接调度中心或者实现执行器。除此之外，还提供了 “多任务模式”和“httpJobHandler”等其他跨语言方案；

32、国际化：调度中心支持国际化设置，提供中文、英文两种可选语言，默认为中文；

33、容器化：提供官方docker镜像，并实时更新推送dockerhub，进一步实现产品开箱即用；

34、线程池隔离：调度线程池进行隔离拆分，慢任务自动降级进入”Slow”线程池，避免耗尽调度线程，提高系统稳定性；

35、用户管理：支持在线管理系统用户，存在管理员、普通用户两种角色；

36、权限控制：执行器维度进行权限控制，管理员拥有全量权限，普通用户需要分配执行器权限后才允许相关操作；

# 2 SpringBoot集成xxl-job执行器

- [xxl-job|2.springboot集成xxl-job执行器](https://blog.csdn.net/qq_39449880/article/details/126004445)

1. “执行器”项目：xxl-job-executor-sample-springboot (提供多种版本执行器供选择，现以 springboot 版本为例，可直接使用，也可以参考其并将现有项目改造成执行器)

2. 作用：负责接收“调度中心”的调度并执行；可直接部署执行器，也可以将执行器集成到现有业务项目中。

## 2.1 添加依赖

```xml
<!--定时器xxljob-->
<dependency>
	<groupId>com.xuxueli</groupId>
	<artifactId>xxl-job-core</artifactId>
	<version>2.0.1</version>
</dependency> 
```

## 2.2 执行器配置

```yaml
xxl:
  job:
    admin:
      address: http://10.71.232.209:7009/xxl-job-admin
    accessToken:
    executor:
      appName: xxx-job
      address:
      ip:
      port: 9999
      logPath: /home/uat/xxx-job
      logretentiondays: 1
```

XXL-JOB执行器的相关配置项的意义，如下所示：

- `xxl.job.admin.addresses` 调度中心的部署地址。若调度中心采用集群部署，存在多个地址，则用逗号分隔。执行器将会使用该地址进行”执行器心跳注册”和”任务结果回调”。
- `xxl.job.executor.appname`执行器的应用名称，它是执行器心跳注册的分组依据。
- `xxl.job.executor.ip`执行器的IP地址，用于”调度中心请求并触发任务”和”执行器注册”。执行器IP默认为空，表示自动获取IP。多网卡时可手动设置指定IP，手动设置IP时将会绑定Host。
- `xxl.job.executor.port`执行器的端口号，默认值为9999。单机部署多个执行器时，注意要配置不同的执行器端口。
- `xxl.job.accessToken`执行器的通信令牌，非空时启用。
- `xxl.job.executor.logpath`执行器输出的日志文件的存储路径，需要拥有该路径的读写权限。
- `xxl.job.executor.logretentiondays`执行器日志文件的定期清理功能，指定日志保存天数，日志文件过期自动删除。限制至少保存3天，否则功能不生效。



执行器支持集群部署，提升调度系统可用性，同时提升任务处理能力。

执行器集群部署时，几点要求和建议：

- 执行器回调地址（xxl.job.admin.addresses）需要保持一致；执行器根据该配置进行执行器自动注册等操作。
- 同一个执行器集群内AppName（xxl.job.executor.appname）需要保持一致；调度中心根据该配置动态发现不同集群的在线执行器列表。

## 2.3 执行器配置类

新建一个执行器配置类，用来读取执行器的配置信息。在config文件夹下新建一个名为 XxlJobConfig 的类，内容如下：

```java
package com.app.config;
 
import com.xxl.job.core.executor.XxlJobExecutor;
import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
 
 
/**
 * xxl-job config
 *
 * @author xuxueli 2017-04-28
 */
@Configuration
@ComponentScan(basePackages = "com.app.job")
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);
 
    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;
 
    @Value("${xxl.job.executor.appname}")
    private String appName;
 
    @Value("${xxl.job.executor.ip}")
    private String ip;
 
    @Value("${xxl.job.executor.port}")
    private int port;
 
    @Value("${xxl.job.accessToken}")
    private String accessToken;
 
    @Value("${xxl.job.executor.logpath}")
    private String logPath;
 
    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;
 
 
    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor  xxlJobSpringExecutor  = new XxlJobSpringExecutor ();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
 
        return xxlJobSpringExecutor ;
    }
 
}
```

## 2.4 编写定时任务

```java
import cn.hutool.core.date.DateUtil;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.handler.annotation.XxlJob;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
 
 
/**
 * XxlJob开发示例（Bean模式）
 *
 * 开发步骤：
 *      1、任务开发：在Spring Bean实例中，开发Job方法；
 *      2、注解配置：为Job方法添加注解 "@XxlJob(value="自定义jobhandler名称", init = "JobHandler初始化方法", destroy = "JobHandler销毁方法")"，注解value值对应的是调度中心新建任务的JobHandler属性的值。
 *      3、执行日志：需要通过 "XxlJobHelper.log" 打印执行日志；
 *      4、任务结果：默认任务结果为 "成功" 状态，不需要主动设置；如有诉求，比如设置任务结果为失败，可以通过 "XxlJobHelper.handleFail/handleSuccess" 自主设置任务结果；
 *
 * @author xuxueli 2019-12-11 21:52:51
 */
 
@Slf4j
@Component
public class JobSample {
    /**
     * 1、简单任务示例（Bean模式）
     */
    @XxlJob(value = "duZhuoJob1")
    public ReturnT<String> startJob2(String param) {
		XxlJobHelper.log(">>>>>>>>>>>>>>>>>" + DateUtil.now() + " param: " + param + ">>>>>>>>>>>>执行");
        System.out.println("duZhuoJob1 is success");
        return ReturnT.SUCCESS;
    }
}
```

## 2.5 调度中心进行配置执行器

配置执行器，AppName要与执行器yml文件中的appname保持一致，名称自选。

对于注册方式，如果调度中心和执行器是本地服务可以选择自动注册

对于发布到服务器的，选择手动录入，机器地址修改为服务器地址，端口号为yml中的执行器port

![img](https://img-blog.csdnimg.cn/f52f75e5dbc3401884083cf291907149.png)

 

![img](https://img-blog.csdnimg.cn/7466803259614682bdf045ef68d5006c.png)

执行器属性说明：

- AppName: 是每个执行器集群的唯一标示AppName, 执行器会周期性以AppName为对象进行自动注册。可通过该配置自动发现注册成功的执行器, 供任务调度时使用;
- 名称: 执行器的名称, 因为AppName限制字母数字等组成,可读性不强, 名称为了提高执行器的可读性;
- 排序: 执行器的排序, 系统中需要执行器的地方,如任务新增, 将会按照该排序读取可用的执行器列表;
- 注册方式：调度中心获取执行器地址的方式；
	- 自动注册：执行器自动进行执行器注册，调度中心通过底层注册表可以动态发现执行器机器地址；
	- 手动录入：人工手动录入执行器的地址信息，多地址逗号分隔，供调度中心使用；
- 机器地址："注册方式"为"手动录入"时有效，支持人工维护执行器的地址信息；

## 2.6 配置任务

 执行器选择**5**中配置的执行器；任务描述可以随意编写；调度类型选择cron，定时执行；运行模式需要选择BEAN模式；JobHandler需要对应上定时任务的注解值

![img](https://img-blog.csdnimg.cn/be2b9d0bd7bb440ea0f2d9c912f9a968.png)

 

![img](https://img-blog.csdnimg.cn/ad1e4dbc8f3f4c61891069a9355c9020.png)

# 3 xxl-job调度器

## 3.1 初始化“调度数据库”

请下载项目源码并解压，获取 “调度数据库初始化SQL脚本” 并执行即可。

“调度数据库初始化SQL脚本” 位置为:

```
/xxl-job/doc/db/tables_xxl_job.sql
```

调度中心支持集群部署，集群情况下各节点务必连接同一个mysql实例;

如果mysql做主从,调度中心集群节点务必强制走主库;

## 3.2 编译源码

解压源码,按照maven格式将源码导入IDE, 使用maven进行编译即可，源码结构如下：

- xxl-job-admin：调度中心

- xxl-job-core：公共依赖

- xxl-job-executor-samples：执行器Sample示例（选择合适的版本执行器，可直接使用，也可以参考其并将现有项目改造成执行器）

	- xxl-job-executor-sample-springboot：Springboot版本，通过Springboot管理执行器，推荐这种方式；

	- xxl-job-executor-sample-frameless：无框架版本；

## 3.3 配置部署“调度中心”

```
调度中心项目：xxl-job-admin作用：统一管理任务调度平台上调度任务，负责触发调度执行，并且提供任务管理平台。
```

#### 步骤一：调度中心配置：

调度中心配置文件地址：

```
/xxl-job/xxl-job-admin/src/main/resources/application.properties
```

调度中心配置内容说明：

```yaml
### 调度中心JDBC链接：链接地址请保持和 2.1章节 所创建的调度数据库的地址一致
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=root_pwd
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
### 报警邮箱
spring.mail.host=smtp.qq.com
spring.mail.port=25
spring.mail.username=xxx@qq.com
spring.mail.password=xxx
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
spring.mail.properties.mail.smtp.socketFactory.class=javax.net.ssl.SSLSocketFactory
### 调度中心通讯TOKEN [选填]：非空时启用；
xxl.job.accessToken=
### 调度中心国际化配置 [必填]： 默认为 "zh_CN"/中文简体, 可选范围为 "zh_CN"/中文简体, "zh_TC"/中文繁体 and "en"/英文；
xxl.job.i18n=zh_CN
## 调度线程池最大线程配置【必填】
xxl.job.triggerpool.fast.max=200
xxl.job.triggerpool.slow.max=100
### 调度中心日志表数据保存天数 [必填]：过期日志自动清理；限制大于等于7时生效，否则, 如-1，关闭自动清理功能；
xxl.job.logretentiondays=30
```

#### 步骤二：部署项目：

如果已经正确进行上述配置，可将项目编译打包部署。

调度中心访问地址：http://localhost:8080/xxl-job-admin (该地址执行器将会使用到，作为回调地址)

默认登录账号 “admin/123456”, 登录后运行界面如下图所示。

至此“调度中心”项目已经部署成功。

#### 步骤三：调度中心集群（可选）：

调度中心支持集群部署，提升调度系统容灾和可用性。

调度中心集群部署时，几点要求和建议：

- DB配置保持一致；
- 集群机器时钟保持一致（单机集群忽视）；
- 建议：推荐通过nginx为调度中心集群做负载均衡，分配域名。调度中心访问、执行器回调配置、调用API服务等操作均通过该域名进行。

#### 其他：Docker 镜像方式搭建调度中心：

- 下载镜像

```bash
# Docker地址：https://hub.docker.com/r/xuxueli/xxl-job-admin/  (建议指定版本号)
docker pull xuxueli/xxl-job-admin
```

- 创建容器并运行

```bash
docker run -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin  -d xuxueli/xxl-job-admin:{指定版本}
/**
* 如需自定义 mysql 等配置，可通过 "-e PARAMS" 指定，参数格式 PARAMS="--key=value  --key2=value2" ；
* 配置项参考文件：/xxl-job/xxl-job-admin/src/main/resources/application.properties
* 如需自定义 JVM内存参数 等配置，可通过 "-e JAVA_OPTS" 指定，参数格式 JAVA_OPTS="-Xmx512m" ；
*/
docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai" -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin  -d xuxueli/xxl-job-admin:{指定版本}
```

# 4 任务详解

## 4.1 配置属性详细说明

### 4.1.1 基础配置：

- 执行器：任务的绑定的执行器，任务触发调度时将会自动发现注册成功的执行器, 实现任务自动发现功能; 另一方面也可以方便的进行任务分组。每个任务必须绑定一个执行器, 可在 "执行器管理" 进行设置;
- 任务描述：任务的描述信息，便于任务管理；
- 负责人：任务的负责人；
- 报警邮件：任务调度失败时邮件通知的邮箱地址，支持配置多邮箱地址，配置多个邮箱地址时用逗号分隔；

### 4.1.2 触发配置：

- 调度类型：
	- 无：该类型不会主动触发调度；
	- CRON：该类型将会通过CRON，触发任务调度；
	- 固定速度：该类型将会以固定速度，触发任务调度；按照固定的间隔时间，周期性触发；
	- 固定延迟：该类型将会以固定延迟，触发任务调度；按照固定的延迟时间，从上次调度结束后开始计算延迟时间，到达延迟时间后触发下次调度；
- CRON：触发任务执行的Cron表达式；
- 固定速度：固定速度的时间间隔，单位为秒；
- 固定延迟：固定延迟的时间间隔，单位为秒；

### 4.1.3 任务配置：

- 运行模式：
	- BEAN模式：任务以JobHandler方式维护在执行器端；需要结合 "JobHandler" 属性匹配执行器中任务；
	- GLUE模式(Java)：任务以源码方式维护在调度中心；该模式的任务实际上是一段继承自IJobHandler的Java类代码并 "groovy" 源码方式维护，它在执行器项目中运行，可使用@Resource/@Autowire注入执行器里中的其他服务；
	- GLUE模式(Shell)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "shell" 脚本；
	- GLUE模式(Python)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "python" 脚本；
	- GLUE模式(PHP)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "php" 脚本；
	- GLUE模式(NodeJS)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "nodejs" 脚本；
	- GLUE模式(PowerShell)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "PowerShell" 脚本；
- JobHandler：运行模式为 "BEAN模式" 时生效，对应执行器中新开发的JobHandler类“@JobHandler”注解自定义的value值；
- 执行参数：任务执行所需的参数；     

### 4.1.4 高级配置：

- 路由策略：当执行器集群部署时，提供丰富的路由策略，包括；
	- FIRST（第一个）：固定选择第一个机器；
	- LAST（最后一个）：固定选择最后一个机器；
	- ROUND（轮询）：；
	- RANDOM（随机）：随机选择在线的机器；
	- CONSISTENT_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。
	- LEAST_FREQUENTLY_USED（最不经常使用）：使用频率最低的机器优先被选举；
	- LEAST_RECENTLY_USED（最近最久未使用）：最久未使用的机器优先被选举；
	- FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度；
	- BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；
	- SHARDING_BROADCAST(分片广播)：广播触发对应集群中所有机器执行一次任务，同时系统自动传递分片参数；可根据分片参数开发分片任务；
- 子任务：每个任务都拥有一个唯一的任务ID(任务ID可以从任务列表获取)，当本任务执行结束并且执行成功时，将会触发子任务ID所对应的任务的一次主动调度。

### 4.1.5 调度过期策略：

- 忽略：调度过期后，忽略过期的任务，从当前时间开始重新计算下次触发时间；
- 立即执行一次：调度过期后，立即执行一次，并从当前时间开始重新计算下次触发时间；
	- 阻塞处理策略：调度过于密集执行器来不及处理时的处理策略；
	- 单机串行（默认）：调度请求进入单机执行器后，调度请求进入FIFO队列并以串行方式运行；
	- 丢弃后续调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，本次请求将会被丢弃并标记为失败；
	- 覆盖之前调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，将会终止运行中的调度任务并清空队列，然后运行本地调度任务；
- 任务超时时间：支持自定义任务超时时间，任务运行超时将会主动中断任务；
- 失败重试次数；支持自定义任务失败重试次数，当任务失败时将会按照预设的失败重试次数主动进行重试；

## 4.2 BEAN模式（方法形式）

Bean模式任务，支持基于方法的开发方式，每个任务对应一个方法。

- 优点：
	- 每个任务只需要开发一个方法，并添加”[@XxlJob](https://github.com/XxlJob)”注解即可，更加方便、快速。
	- 支持自动扫描任务并注入到执行器容器。
- 缺点：要求Spring容器环境；

> 基于方法开发的任务，底层会生成JobHandler代理，和基于类的方式一样，任务也会以JobHandler的形式存在于执行器任务容器中。

### 步骤一：执行器项目中，开发Job方法：

1. 任务开发：在Spring Bean实例中，开发Job方法；
2. 注解配置：为Job方法添加注解 "@XxlJob(value="自定义jobhandler名称", init = "JobHandler初始化方法", destroy = "JobHandler销毁方法")"，注解value值对应的是调度中心新建任务的JobHandler属性的值。
3. 执行日志：需要通过 "XxlJobHelper.log" 打印执行日志；
4. 任务结果：默认任务结果为 "成功" 状态，不需要主动设置；如有诉求，比如设置任务结果为失败，可以通过 "XxlJobHelper.handleFail/handleSuccess" 自主设置任务结果；

### 步骤二：调度中心，新建调度任务

参考上文“配置属性详细说明”对新建的任务进行参数配置，运行模式选中 “BEAN模式”，JobHandler属性填写任务注解“@XxlJob”中定义的值；

![输入图片说明](https://img-blog.csdnimg.cn/img_convert/bc8a53f6f7579c673e01bfaac83c2b92.png)

### 原生内置Bean模式任务

为方便用户参考与快速实用，示例执行器内原生提供多个Bean模式任务Handler，可以直接配置实用，如下：

- demoJobHandler：简单示例任务，任务内部模拟耗时任务逻辑，用户可在线体验Rolling Log等功能；

- shardingJobHandler：分片示例任务，任务内部模拟处理分片参数，可参考熟悉分片任务；

- httpJobHandler：通用HTTP任务Handler；业务方只需要提供HTTP链接等信息即可，不限制语言、平台。示例任务入参如下：

	```
	url: http://www.xxx.com
	method: get 或 post
	data: post-data
	```

- commandJobHandler：通用命令行任务Handler；业务方只需要提供命令行即可；如 “pwd”命令；

## 4.3 GLUE模式(Java)

任务以源码方式维护在调度中心，支持通过Web IDE在线更新，实时编译和生效，因此不需要指定JobHandler。开发流程如下：

### 步骤一：调度中心，新建调度任务：

参考上文“配置属性详细说明”对新建的任务进行参数配置，运行模式选中 “GLUE模式(Java)”；

![输入图片说明](https://img-blog.csdnimg.cn/img_convert/05d28b547e7d3b15229d28b7446c64d4.png)

### 步骤二：开发任务代码：

选中指定任务，点击该任务右侧“GLUE”按钮，将会前往GLUE任务的Web IDE界面，在该界面支持对任务代码进行开发（也可以在IDE中开发完成后，复制粘贴到编辑中）。

版本回溯功能（支持30个版本的版本回溯）：在GLUE任务的Web IDE界面，选择右上角下拉框“版本回溯”，会列出该GLUE的更新历史，选择相应版本即可显示该版本代码，保存后GLUE代码即回退到对应的历史版本；

![输入图片说明](https://img-blog.csdnimg.cn/img_convert/ff876ff8426ed31c08d06c490bee1add.png)

## 4.4 GLUE模式(Shell)

### 步骤一：调度中心，新建调度任务

参考上文“配置属性详细说明”对新建的任务进行参数配置，运行模式选中 “GLUE模式(Shell)”；

### 步骤二：开发任务代码：

选中指定任务，点击该任务右侧“GLUE”按钮，将会前往GLUE任务的Web IDE界面，在该界面支持对任务代码进行开发（也可以在IDE中开发完成后，复制粘贴到编辑中）。

该模式的任务实际上是一段 “shell” 脚本；

![输入图片说明](https://img-blog.csdnimg.cn/img_convert/36f87c30f03bf54ad197b35e2d53873f.png)

- [使用xxl-job执行GLUE(shell)](https://blog.csdn.net/fsjwin/article/details/110980093)

### 操作解析：

需求：使用xxl-job实现调用脚本的功能，实现目的是在xxl-job控制器上创建一个test文件既可以了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201210204828506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Zzandpbg==,size_16,color_FFFFFF,t_70)

## 4.5 GLUE模式(Python)

### 步骤一：调度中心，新建调度任务

参考上文“配置属性详细说明”对新建的任务进行参数配置，运行模式选中 “GLUE模式(Python)”；

### 步骤二：开发任务代码：

选中指定任务，点击该任务右侧“GLUE”按钮，将会前往GLUE任务的Web IDE界面，在该界面支持对任务代码进行开发（也可以在IDE中开发完成后，复制粘贴到编辑中）。

该模式的任务实际上是一段 “python” 脚本；

![输入图片说明](https://img-blog.csdnimg.cn/img_convert/0e44b3cac2134801b992e7a7894d3e83.png)

## 4.6 GLUE模式(NodeJS)

### 步骤一：调度中心，新建调度任务

参考上文“配置属性详细说明”对新建的任务进行参数配置，运行模式选中 “GLUE模式(NodeJS)”；

### 步骤二：开发任务代码：

选中指定任务，点击该任务右侧“GLUE”按钮，将会前往GLUE任务的Web IDE界面，在该界面支持对任务代码进行开发（也可以在IDE中开发完成后，复制粘贴到编辑中）。

该模式的任务实际上是一段 “nodeJS” 脚本；

# 附录A 其他参考博文

- [02 xxl-job执行器启动流程](https://juejin.cn/post/6937669199942647845)
- [03 xxl-job任务执行流程](https://juejin.cn/post/6938034809197297694)
- [xxl-job 编写定时任务](https://juejin.cn/post/7001961673154117646)
