- [linux shell 当前时间、过去、未来几天的时间设置-CSDN博客](https://blog.csdn.net/sodaloveer/article/details/124866941)

# 一、date 命令含义

date命令的功能是显示和设置系统日期和时间。
该命令的一般格式为： date [选项] 显示时间格式（以+开头，后面接格式）
命令中各选项的含义分别为：
-d datestr, --date datestr 显示由datestr描述的日期(非系统时间)
-s datestr, --set datestr 将系统时间设为 datestr 中所设定的时间
-u, --universal 显示或设置通用时间(目前的格林威治时间)
时间域
% H 小时（00…23）
% I 小时（01…12）
% k 小时（0…23）
% l 小时（1…12）
% M 分（00…59）
% p 显示出AM或PM
% r 时间（hh：mm：ss AM或PM），12小时
% s 从1970年1月1日00：00：00到目前经历的秒数
% S 秒（00…59）
% T 时间（24小时制）（hh:mm:ss）
% X 显示时间的格式（％H:％M:％S）
% Z 时区 日期域
% a 星期几的简称（ Sun…Sat）
% A 星期几的全称（ Sunday…Saturday）
% b 月的简称（Jan…Dec）
% B 月的全称（January…December）
% c 日期和时间（ Mon Nov 8 14：12：46 CST 1999）
% d 一个月的第几天（01…31）
% D 日期（mm／dd／yy）
% h 和%b选项相同
% j 一年的第几天（001…366）
% m 月（01…12）
% w 一个星期的第几天（0代表星期天）
% W 一年的第几个星期（00…53，星期一为第一天）
% x 显示日期的格式（mm/dd/yy）
% y 年的最后两个数字（ 1999则是99）
% Y 年（例如：1970，1996等）
需要特别说明的是，只有超级用户才能用date命令设置时间，一般用户只能用date命令显示时间。

```
以上的内容来源于网络资料
```

# 二、date 使用命令

## 1.提取日期元素

### 1.年月日

> 代码如下（示例）：

```bash
date +%Y     //year,输出4位数年
date +%y     //输出2位数的年
date +%m     //m表示month，输出的结果是月份
date +%d     //d表示day,日
date +%D     //D表示特殊格式的日期表示方法
```

```bash
date  +%Y%m%d  //输出年月日
date  +%F     //输出的年月日会用连字符隔开
```

### 2.时分秒

> 代码如下（示例）：

```bash
 date +%H   //H表示hour小时
 date +%M   //M表示minute分钟
 date +%S   //S表示second秒
 date +%s   //输出结果是一个时间戳，表示距离1970年1月1日0时0分过去多少秒
 date +%T   //以下面格式输出时分秒
```

```bash
 date +%H:%M:%S   //效果跟上条命令 date +%T 一样
```

### 3.星期

> 代码如下（示例）：

```bash
date +%w    //输出结果为星期几，比如下图表示当前系统时间是星期五
```

> 代码如下（示例）：

```bash
date +%W   //表示当前为今年的第几周，比如我这里为今年的第12周
```

### 4.时间戳

> 代码如下（示例）：

```bash
date +%s          //获取当前时间戳
```

> 代码如下（示例）：

```bash
date +%s -d "2022-05-20 10:07:38"  //将日期换算成时间戳
```

> 代码如下（示例）：

```bash
//赋值也可以
//求时间戳yyyy-mm-dd,yyyymmdd,mm/dd/yyyy格式都可以
cur_day=2022-05-12 
date +%s -d "$cur_day"
cur_day=20220512 
date +%s -d "$cur_day"
cur_day=05/12/2022
date +%s -d "$cur_day"
```

### 5.日历

> 代码如下（示例）：

```bash
 cal     //展示日历
```

## 2、date日期时间操作

### 1.[获取当前时间](https://so.csdn.net/so/search?q=获取当前时间&spm=1001.2101.3001.7020)

> 代码如下（示例）：

```bash
//当前的时间：年月日 时分秒
current_time=$(date "+%Y%m%d %H%M%S")
echo $current_time
//!!!!!!!!!!注意区分大小写，%m表示天，%M表示分
//格式可以自己设置
current_time=$(date "+%Y-%m-%d %H:%M:%S")
echo $current_time
//效果跟上条命令一样
current_time=$(date "+%F %T")
echo $current_time
```

### 2.获取相对日期

```
例如：前1个小时、后一个小时、前天、昨天、明天、后天，上个月、下个月时间
```

-d 选项还可以告诉您，相对于当前日期若干天的究竟是哪一天，从现在开始的若干天或若干星期以后，或者以前（过去）。通过将这个相对偏移使用引号括起来，作为 -d 选项的参数，就可以完成这项任务。

> 代码如下（示例）：

```bash
 //获取1个小时前的日期
date -d "1 hour ago" +"%Y-%m-%d %H:%M"
date -d "-1 hour" +"%Y-%m-%d %H:%M"
date -d "last hour"  +"%Y-%m-%d %H:%M"

 //获取前天的日期
date -d "-2 day" "+%Y-%m-%d" 
date -d "2 days ago" "+%Y-%m-%d"

//获取昨天的日期
date -d "-1 day" "+%Y-%m-%d"  
date -d "1 days ago" "+%Y-%m-%d"
date -d "yesterday" "+%Y-%m-%d"
date -d "last-day" "+%Y-%m-%d"

// 获取具体日期的前几天
// 规则1：date -d "n days ago $具体日期" "+%Y-%m-%d " 
date -d"30 day ago 2022-05-20" "+%Y-%m-%d"
date -d"30 day ago 20220520" "+%Y%m%d"
cur_day=20220517
date -d"30 day ago $cur_day" "+%Y%m%d"
//规则2：date -d "-n days $具体日期" "+%Y-%m-%d " 
date -d"-30 day 20220520" "+%Y%m%d"
cur_day=20220517
date -d"-30 day $cur_day" "+%Y%m%d"

 //获取1个小时后的日期
date -d "+1 hour" +"%Y-%m-%d %H:%M"
date -d "next hour"  +"%Y-%m-%d %H:%M"

 //获取2个小时后的日期
date -d "+2 hours" +"%Y-%m-%d %H:%M"

//查看明天日期 "+%Y-%m-%d"格式可以自己调整
date -d "1 day" "+%Y-%m-%d" 
date -d "next day" "+%Y-%m-%d"
date -d "tomorrow" "+%Y-%m-%d"

//查看两天后日期
date -d "2 day" "+%Y-%m-%d"
date -d "+2 day" "+%Y-%m-%d"

//查看一个月之前的日期
date -d "-1 month" "+%Y-%m-%d"
date -d "last-month" "+%Y-%m-%d"

//查看一个月之后的日期
date -d "+1 month" "+%Y-%m-%d"
date -d "next-month" "+%Y-%m-%d"

//查看明年日期
date -d "+1 year" "+%Y-%m-%d"
date -d "next-year" "+%Y-%m-%d"
```

### 3.日期加减计算

#### （1）求两个日期之间的差

先转化为时间戳，求间隔的秒数，然后再求对应的时分天

> 代码如下（示例）：

```bash
// 求几个几天
t1='2021-05-13'
t2='2021-05-20'
diff=$(((`date -d "$t2" '+%s'` - `date -d   "$t1" '+%s' ` ) / 86400))
echo $diff
```

------

# 三、 总结

> 代码如下（示例）：

```c
## 昨天（（上一 ）小时/星期/月/年）的日期
### 使用 n hour/week/day/month/year/ ago 
### 使用 - n hour/week/day/month/year/
### 使用 last hour/week/day/month/year/
### 使用 yesterday(昨天）

## 明天（（下一 ）星期/月/年）的日期
### 使用 + n hour/week/day/month/year/ 
### 使用 next hour/week/day/month/year/
### 使用 tomorrow(明天）

## 过去及未来2天(星期/月/年）及以上的日期
###  使用 n hours/weeks/days/months/years/ ago指令可以得到过去的日期
### 使用- n hours/weeks/days/months/years/ 指令可以得到过去的日期
### 使用+ n hours/weeks/days/months/years/  指令可以得到未来的日期
```