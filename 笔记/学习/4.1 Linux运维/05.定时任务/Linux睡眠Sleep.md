# 一、介绍

在 `Linux` 的 `bash` 中，利用 `sleep` 和 `usleep` 命令可以控制睡眠时长，进行延时操作。

sleep命令将下一个命令的调用进程挂起一段指定的时间。当以下命令的执行取决于前一个命令的成功完成时，此属性很有用。

- sleep：默认以秒为单位
- usleep：默认以微秒为单位（1s = 1000ms = 1000000us）

具体例子为：

```shell
sleep 1 # 睡眠1s

sleep 1s # 睡眠1s 

sleep 1m # 睡眠1分钟

sleep 1h # 睡眠1小时

sleep 1d # 睡眠1天
```

有时候，`usleep` 无法使用，可以使用以下格式进行睡眠：

```shell
sleep 0.01 # 10ms

sleep 0.1 # 100ms
```

# 二、语法

默认情况下，系统读取之后 `sleep` 的数字作为秒数。

```shell
sleep [number]

sleep 1h 2m 0.5s
```

`sleep` 命令接受浮点数，允许多个值相加计算睡睡眠时长。可使用单位有：

- s 秒
- m 分钟
- h 小时
- d 天

需要中断 `sleep` 命令可通过 `Ctrl + C` 实现。

```shell
# 查看帮助
sleep --help

# 版本信息
$ sleep --version
sleep (GNU coreutils) 8.25
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Written by Jim Meyering and Paul Eggert.
```

# 三、实操

## 1.设置警报

一定时间后播放文件。

```shell
sleep 8h 30m 50s && mplayer alarm.mp3
```

## 2.终端中的睡眠命令

```shell
$ sleep 1 && echo "one" && sleep 1 && echo "two"
one # 间隔1s
two
```

## 3.变量分配给sleep

可以将变量分配给 `sleep` 命令。

```shell
# 编写shell
#!/bin/bash
SLEEP_INTERVAL="5"
CURRENT_TIME=$(date +"%T")
echo "Time before sleep: ${CURRENT_TIME}"
echo "Sleeping for ${SLEEP_INTERVAL} secondes"
sleep ${SLEEP_INTERVAL}
CURRENT_TIME=$(date +"%T")
echo "Time after sleep: ${CURRENT_TIME}"

# 执行
Hider@Hider1214 MINGW64 /e
$ sh run.sh
Time before sleep: 22:55:57
Sleeping for 5 secondes
Time after sleep: 22:56:02
```

## 4.定义检查间隔

利用 `ping` 命令检查网站是否在线。

```shell
# 编写shell
#!/bin/bash
while :
  do
    if ping -n 1 www.baidu.com &> /dev/null
    then
  echo "Baidu is online"
  break
  fi
  sleep 10
done
```