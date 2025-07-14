# 一、Crontab

Crontab命令用于设置周期性被执行的指令。该命令从标准输入设备读取指令，并将其存放于“crontab”文件中，以供之后读取和执行。

可以使用Crontab定时处理离线任务，比如每天凌晨2点更新数据等，经常用于系统任务调度。

## 1.1 服务启动和关闭

一般Linux系统中都会装有Crontab，如果没有安装可以使用包管理工具安装：

```bash
yum -y install vixie-cronyum -y install crontabs
```

Crontab服务的启动和关闭命令如下：

```bash
service crond start 
# 启动服务
service crond stop  
# 关闭服务
service crond restart   
# 重启服务
service crond reload    
# 重新载入配置
service crond status    
# 查看crontab服务状态
```

## 1.2 **任务调度全局配置**

crontab全局任务调度配置在如下的目录：

```bash
cron.d/       cron.daily/   cron.deny     cron.hourly/  cron.monthly/ crontab       cron.weekly/
```

- cron.daily是每天执行一次的job

- cron.weekly是每个星期执行一次的job

- cron.monthly是每月执行一次的job

- cron.hourly是每个小时执行一次的job

- cron.d是系统自动定期需要做的任务

- crontab是设定定时任务执行文件

- cron.deny文件就是用于控制不让哪些用户使用Crontab的功能

## 1.3 用户配置文件

每个用户都有自己的crontab配置文件，使用crontab -e命令进行编辑。保存后系统会自动存放与/var/spool/cron/目录中，文件以用户名命名。

linux的crontab服务每隔一分钟去读取一次/var/spool/cron，/etc/crontab，/etc/cron.d下面所有的内容，crontab命令一览：

- crontab -e: 编辑当前用户的定时任务列表

- crontab -l: 查看当前用户的定时任务列表

- crontab -r: 删除当前用户的定时任务列表

## 1.4 crontab定时任务格式

crontab每一条记录为一个定时任务，定时人遵循相应的定义规则。

![img](https://img-blog.csdnimg.cn/img_convert/0951973f4b4dcac0c4bfda8110a615a7.png)

其中前面的6个星号表示的含义如下：

**内容含义说明：**

| 项目          | 含义                 | 范围                                       |
| :------------ | :------------------- | :----------------------------------------- |
| 第一个”*“     | 一小时当中的第几分钟 | 0-59                                       |
| 第二个”*“     | 一天当中的第几小时   | 0-23                                       |
| 第三个”*“     | 一个月当中的第几天   | 1-31                                       |
| 第四个”*“     | 一年当中的第几月     | 1-12                                       |
| 第五个”*“     | 一周当中的星期几     | 0-7（0和7都代表星期日）                    |
| 第六个command | 要执行的命令         | 可以是系统命令，也可以是自己编写的脚本文件 |

**每一个星号部分可用下面的特殊符号：**

| 特殊字符 | 代表意义                                                     |
| :------- | :----------------------------------------------------------- |
| *        | 代表任何时刻都接受的意思！举例来说，范例一内那个日、月、周都是 * ，就代表著 **不论何月、何日的礼拜几的 12:00 都运行后续命令 ** 的意思！ |
| ,        | 代表分隔时段的意思。举例来说，如果要下达的工作是 3:00 与 6:00 时，就会是：0 3,6 * * * command 时间参数还是有五栏，不过第二栏是 3,6 ，代表 3 与 6 都适用！（不能加空格） |
| -        | 代表一段时间范围内，举例来说， 8 点到 12 点之间的每小时的 20 分都进行一项工作：20 8-12 * * * command 仔细看到第二栏变成 8-12 喔！代表 8,9,10,11,12 都适用的意思！ |
| /n       | 那个 n 代表数字，亦即是『每隔 n 单位间隔』的意思，例如每五分钟进行一次，则：*/5 * * * * command 很简单吧！用 * 与 /5 来搭配，也可以写成 0-59/5 ，相同意思！ |

## 1.5  Crontab命令

### 1.5.1 命令格式

```powershell
crontab [-u user] file
crontab [-u user] [ -e | -l | -r ]
```

### 1.5.2 命令功能

通过crontab 命令，我们可以在固定的间隔时间执行指定的系统指令或 shell script脚本。时间间隔的单位可以是分钟、小时、日、月、周及以上的任意组合。这个命令非常适合周期性的日志分析或数据备份等工作。

### 1.5.3 命令参数

```bash
-u user：用来设定某个用户的crontab服务，例如，“-u ixdba”表示设定ixdba用户的crontab服务，此参数一般有root用户来运行。
file：file是命令文件的名字,表示将file做为crontab的任务列表文件并载入crontab。如果在命令行中没有指定这个文件，crontab命令将接受标准输入（键盘）上键入的命令，并将它们载入crontab。
-e：编辑某个用户的crontab文件内容。如果不指定用户，则表示编辑当前用户的crontab文件。
-l：显示某个用户的crontab文件内容，如果不指定用户，则表示显示当前用户的crontab文件内容。
-r：从/var/spool/cron目录中删除某个用户的crontab文件，如果不指定用户，则默认删除当前用户的crontab文件。
-i：在删除用户的crontab文件时给确认提示。
```

# 二、crontab定时任务实例

## 2.1 时间频次示例

注意，一般最好不要同时采用几号和每周几，可能会出现混淆；

**时间表达式示例：**

| 时间                                            | 含义                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| 10 * * * * 命令                                 | 每小时的第10分钟执行一次命令                                 |
| 45 22 * * * 命令                                | 在22点45分执行命令                                           |
| 0 17 * * 1 命令                                 | 每周1的17点0分执行命令                                       |
| 0 5 1,15 * * 命令                               | 每月1日和15日的凌晨5点0分执行命令                            |
| 40 4 * * 1-5 命令                               | 每周一到周五的凌晨4点40分执行命令                            |
| */10 4 * * * 命令                               | 每天的凌晨4点，每隔10分钟执行一次命令                        |
| 0 0 1,15 * 1 命令                               | 每月1日和15日，每周1的0点0分都会执行命令。 日和星期不是并且的关系，是或者的关系。 |
| 0 */10 * * * /usr/bin/curl http://www.baidu.com | 每10分钟定时请求一个地址                                     |
| 10 14 1 * * sh test.sh                          | 每个月的1号 14:10 运行                                       |
| */1 * * * * sh test.sh                          | 每分钟运行一次命令                                           |
| 59 23 * * 1,2,3,4,5 sh test.sh                  | 每个工作日23:59都进行备份作业。                              |
| 59 23 * * 1-5 sh test.sh                        | 每个工作日23:59都进行备份作业。                              |
| 10 0 * * * sh test.sh                           | 在 凌晨00:10运行                                             |

## 2.2 输入输出重定向

在上面更新数据定时任务的实例中，如果执行update.sh的时候有错误信息输出，会输出到哪儿呢？

在没有配置输出重定向的时候，定时任务会见错误输出到下面的文件：/var/log/mail/{$user}，这是很不好的，我们往往需要把不同的脚本输出到不同的日志文件，方便查看脚本的执行情况，这个时候需要使用输出重定向。

```bash
58 23 * * * sh /home/work/update.sh >> /home/work/log/update.log 2>&1
```

Linux中使用0-3作为文件描述符来表示标准流。

| **名称**            | **类型**              | **文件描述符** | **操作** |
| ------------------- | --------------------- | -------------- | -------- |
| stdin 标准输入      | standard input        | 0              | <,<<     |
| stdout 标准输出     | standard output       | 1              | >,>>     |
| stderr 标准错误输出 | standard error output | 2              | 2>,2>>   |

在上面的定时任务脚本中，update.sh后面的>>表示将输出追加到update.log文件中，2>&1表示标准错误输出重定向等同于标准输出。

**正确、错误日志的输出是否写入到文件方法：**

1. 不输出任何内容（建议使用方法一）

```bash
*/1 * * * * /root/XXXX.sh >/dev/null 2>&1 
或
*/1 * * * * /root/XXXX.sh &>/dev/null    //&表示任何内容
```

2. 将正确和错误日志都输出到 /tmp/load.log

```bash
*/1 * * * * /root/XXXX.sh > /tmp/load.log 2>&1
```

3. 只输出正确日志到 /tmp/load.log

```bash
*/1 * * * * /root/XXXX.sh > /tmp/load.log
或
*/1 * * * * /root/XXXX.sh 1> /tmp/load.log    //1可以省略
```

4. 只输出错误日志到 /tmp/load.log

```bash
*/1 * * * * /root/XXXX.sh 2> /tmp/load.log
```

部分解释:

```bash
/dev/null 代表空设备文件
> 代表重定向到哪里
1 表示stdout标准输出，系统默认值是1，所以">/dev/null"等同于"1>/dev/null"
2 表示stderr标准错误
& 表示等同于的意思，2>&1，表示2的输出重定向等同于1
```

## 2.3 忽略输入nohup

有时候我们还会看到在定时任务脚本前面有一个nohup的命令，该命令用于指定后面的脚本忽略输入。

```bash
58 23 * * * nohup sh /home/work/update.sh >> /home/work/log/update.log 2>&1
```

## 2.4 后台执行&

无论是在控制台直接执行，还是在定时任务脚本中，我们在某一条命令的最后面加上&符号，表示当前命令在后台运行，不占用控制台。

```bash
58 23 * * * nohup sh /home/work/update.sh >> /home/work/log/update.log 2>&1 &
```

这里需要注意，如果执行的命令有等待输入的交互，把这个命令使用&放在后台运行时，它会一直等待输入，但是有没有输入，就卡住不动了。

## 2.5 时间处理

再回顾上面的更新任务，它是将每一天的运行结果都重定向到update.log文件，如果输出比较多，我们就需要每一天输出的不同的文件，可以通过在输出目录中指定日期用来分割。如下：

```bash
58 23 * * * nohup sh /home/work/update.sh >> /home/work/log/`update_"date +\%Y\%m\%d".log` 2>&1 &# 或者下面的形式58 23 * * * nohup sh /home/work/update.sh >> "/home/work/log/update_"date +\%Y\%m\%d".log" 2>&1 &
```

这里使用date函数，需要注意的是，crontab里面的脚本命令和直接在命令行运行的脚本可能是不一样的，尤其是date函数中的%需要转义。

可以直接在命令行运行下面命令：

```bash
sh /home/work/update.sh >> /home/work/log/update_`date +"%Y%m%d".log` 2>&1 &
```

达到和定时任务里面相同的效果，都是输出到update_20201120.log的文件。

但是如果直接在crontab中这样配置，则无法识别，不会执行该定时任务。下面的crontab任务是不会执行的，date函数将会报错。

```bash
58 23 * * * nohup sh /home/work/update.sh >> /home/work/log/`update_"date +%Y%m%d".log` 2>&1 &
```

# 附录A

## A.1 注意事项

1. 新创建的cron job，不会马上执行，至少要过2分钟才执行。如果重启cron则马上执行

2. 当crontab突然失效时，可以尝试/etc/init.d/crond restart解决问题。或者查看日志看某个job有没有执行/报错tail -f /var/log/cron。

3. 千万别乱运行crontab -r。它从Crontab目录（/var/spool/cron）中删除用户的Crontab文件。删除了该用户的所有crontab都没了。

## A.2 参考博客

- [『学了就忘』Linux系统定时任务](https://www.cnblogs.com/liuyuelinfighting/p/15719096.html)