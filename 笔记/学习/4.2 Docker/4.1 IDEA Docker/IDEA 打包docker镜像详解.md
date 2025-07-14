- [IDEA 打包docker镜像详解_Small_Mayi的博客-CSDN博客_idea打包镜像](https://blog.csdn.net/qq_33411202/article/details/102834385)

##### 1.下载[docker](https://so.csdn.net/so/search?q=docker&spm=1001.2101.3001.7020) 插件

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019103111174831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDExMjAy,size_16,color_FFFFFF,t_70)

##### 2.配置连接docker

Edit Configurations 添加Dockerfile

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031112133780.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031112226566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDExMjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031113941590.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDExMjAy,size_16,color_FFFFFF,t_70)
后面那个是容器端口

##### 3.编写Dockerfile

放在项目根目录

```csharp
FROM java:8
#作者
MAINTAINER xu
#声明一个挂载点，容器内此路径会对应宿主机的某个文件夹
VOLUME /tmp
#复制上下文目录下的target/demo-1.0.0.jar 到容器里
COPY target/demo-0.0.1.jar demo-0.0.1.jar

#bash方式执行，使demo-0.0.1jar可访问
#RUN新建立一层，在其上执行这些命令，执行结束后， commit 这一层的修改，构成新的镜像。
RUN bash -c "touch /demo-0.0.1.jar"

#声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务
EXPOSE 8899

#指定容器启动程序及参数   <ENTRYPOINT> "<CMD>"
ENTRYPOINT ["java","-jar","demo-0.0.1.jar"]
```

##### 5.运行

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031122026500.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031122137914.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDExMjAy,size_16,color_FFFFFF,t_70)
镜像已成功运行，浏览器访问即可。

##### 其他

docker基本操作

```csharp
#查看镜像
docker images
#删除镜像
docker rmi 镜像id或name

#查看所有容器，包括未运行
docker ps -a 
#停止容器
docker stop 容器id或name
#启动容器
docker start 容器id或name
#删除容器
docker rm 容器id或name
```