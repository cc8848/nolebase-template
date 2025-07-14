- [Hive_数仓分层理论及命名规范_大数据翻身的博客-CSDN博客_hive 字段命名规则](https://blog.csdn.net/qq_56870570/article/details/119807224?spm=1001.2014.3001.5502)

## **1. 数仓分层框架**

![img](https://img-blog.csdnimg.cn/20210818095521596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)

**ODS层：**原始数据层，存放原始数据，直接加载原始日志、数据，数据保持原貌不做处理。

**DWD层：**对ODS层数据进行清洗（去除空值，脏数据，超过极限范围的数据）、脱敏等。保存业务事实明细，一行信息代表一次业务行为，例如一次下单。

**DIM层：**维度层，保存维度数据，主要是对业务事实的描述信息，例如何人，何时，何地等。

**DWS层：**以DWD为基础，按天进行轻度汇总。一行信息代表一个主题对象一天的汇总行为，例如一个用户一天下单次数（按天）。

**DWT层：**以DWS为基础，对数据进行累积汇总。一行信息代表一个主题对象的累积行为，例如一个用户从注册那天开始至今一共下了多少次单（按月、年）。

**ADS层：**为各种统计报表提供数据。

## 2. 数据仓库为什么要分层

- **把复杂问题简单化：**将复杂的任务分解成多层来完成，每一层只处理简单的任务，方便定位问题。
- **减少重复开发：**规范数据分层，通过的中间层数据，能够减少极大的重复计算，增加一次计算结果的复用性。
- **隔离原始数据：**不论是数据的异常还是数据的敏感性，使真实数据与统计数据隔离开。

## **3. 数仓命名****规范

### **3.1 表命名**

- ODS层命名为ods_表名
- DIM层命名为dim_表名
- DWD层命名为dwd_表名
- DWS层命名为dws_表名  
- DWT层命名为dwt_表名
- ADS层命名为ads_表名
- 临时表命名为tmp_表名

**或 库.表**

如：

- ods.表名
- dim.表名
- dwd.表名
- ......

**或 {project_name}.ods_{数据来源}_{源系统表名}**

如：

- 增量数据： {project_name}.ods_{数据来源}_{源系统表名}_delta
- 全量数据： {project_name}.ods_{数据来源}_{源系统表名}

数据来源说明：
01 -> hdfs 数据
02 -> mysql 数据
03 -> redis 数据
04 -> mongodb 数据
05 -> tidb 数据

举例如下：
行为日志表： ods_01_action_log（ods_hdfs_action_log）
用户表： ods_02_user（ods_mysql_user）

### **3.2 脚本命名**

- 数据源_to_目标_db/log.sh
- 用户行为脚本以log为后缀；业务数据脚本以db为后缀。

### **3.3 表字段类型**

- 数量类型为bigint
- 金额类型为decimal(16, 2)，表示：16位有效数字，其中小数部分2位
- 字符串(名字，描述信息等)类型为string
- 主键外键类型为string
- 时间戳类型为bigint

## 4. 数据集市与数仓(引入)

数据集市（Data Market）：是一种微型的数据仓库，它通常有更少的数据，更少的主题区域，以及更少的历史数据，因此是部门级的，一般只能为某个局部范围内的管理人员服务。

数据仓库（Data Warehouse）：数据仓库是企业级的，能为整个企业各个部门的运行提供决策支持手段。

![img](https://img-blog.csdnimg.cn/20210818101922587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzU2ODcwNTcw,size_16,color_FFFFFF,t_70)