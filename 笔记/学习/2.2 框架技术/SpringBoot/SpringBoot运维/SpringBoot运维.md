# nohub命令和java -jar启动Spring Boot

## 1.nohub命令启动Spring Boot项目

```bash
nohup java -Xms1024m -Xmx2048m -jar demo-0.0.1-SNAPSHOT.jar 1>start.out 2>&1 &
```

## 2.java -jar启动Spring Boot项目

java -Xms800m -Xmx800m -XX:PermSize=256m -XX:MaxPermSize=512m -XX:MaxNewSize=512m -jar demo-0.0.1-SNAPSHOT.jar > rocketmq-console-log.file 2>&1 &

1、-Xms ：表示java虚拟机堆区内存初始内存分配的大小，通常为操作系统可用内存的1/64大小即可，但仍需按照实际情况进行分配。有可能真的按照这样的一个规则分配时，设计出的软件还没有能够运行得起来就挂了。

2、-Xmx： 表示java虚拟机堆区内存可被分配的最大上限，通常为操作系统可用内存的1/4大小。但是开发过程中，通常会将 -Xms 与 -Xmx两个参数的配置相同的值，其目的是为了能够在java垃圾回收机制清理完堆区后不需要重新分隔计算堆区的大小而浪费资源。

3、-XX:newSize：表示新生代初始内存的大小，应该小于 -Xms的值；

4、-XX:MaxnewSize：表示新生代可被分配的内存的最大上限；当然这个值应该小于 -Xmx的值；

5、-Xmn：至于这个参数则是对 -XX:newSize、-XX:MaxnewSize两个参数的同时配置，也就是说如果通过-Xmn来配置新生代的内存大小，那么-XX:newSize = -XX:MaxnewSize = -Xmn，虽然会很方便，但需要注意的是这个参数是在JDK1.4版本以后才使用的。

6、-XX:PermSize：表示非堆区初始内存分配大小，其缩写为permanent size（持久化内存）

7、-XX:MaxPermSize：表示对非堆区分配的内存的最大上限。

## 3.nohub和java -jar的启动区别

### **方法一：直接启动 jar 包**

```cobol
java -jar XXX.jar
```

当前ssh窗口会被锁定，不能再操作其他命令，如果需要执行其他命令，则需要打断进程，可按CTRL + C打断程序运行，或直接关闭窗口，程序也会退出。

### **方法二：后台启动 jar 包**

```cobol
java -jar XXX.jar &
```

&代表在后台运行。
缺点：当前ssh窗口不被锁定，但是当窗口关闭时，程序中止运行。

### **方法三：后台不挂断启动**

```cobol
nohup java -jar XXX.jar  
```

用 nohub 命令启动 jar 包，是在后台不挂断运行，关闭终端窗口或者 CTRL + C 命令也不会终止程序。
**当用 nohub命令启动 jar 包的时候，如果不指定日志输出文件，则所有的输出都会被重定向到 nohub.out 的文件中。**

### **方式四：指定日志输出的启动**

```cobol
nohup java -jar XXX.jar >log.out &
```

通过 >log.out 命令，将程序的日志输出重定向到 log.out 文件中。如果不指定日志输出文件，则会输出到 nohub 命令默认的输出文件，nohub.out 文件中。

### 方式五**：指定配置文件启动**

```cobol
nohup java -jar XXX.jar --spring.profiles.active=pro >log.out &
```

通过 --spring profiles.active=pro 来指定，启动的的是application-pro.yml 或者 application-pro.properties 配置文件。

### 方式六：指定分配系统资源大小

```cobol
nohup java -Xms500m -Xmx500m -jar XXX.jar --spring.profiles.active=dev >log.out &
```