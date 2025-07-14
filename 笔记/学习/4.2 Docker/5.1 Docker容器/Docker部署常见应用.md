# 1 Docker安装LskyPro

> 参考资料：
>
> - [基于 Docker 安装之 lsky-pro_v2 图床 | 小康博客 (antmoe.com)](https://www.antmoe.com/posts/ebff2/)
> - [Typora+PicGo+LskyPro打造舒适写作环境 - IT王小二 - 博客园 (cnblogs.com)](https://www.cnblogs.com/itwxe/p/15132289.html)

```bash
docker run -d --name=lskypro -p 4080:80 -v /itwxe-hk/dockerData/lskypro:/var/www/html itwxe/lskypro:1.6.3
```

tips：

- 此处遇到个坑，配置数据库地址的时候，需要查看本地IP地址，不能直接使用外网地址或者127.0.0.1，经查看，是PHP用Docker部署的问题。

> 参考系列博文：
>
> - [佛西博客 - Docker搭建lskypro兰空图床](https://foxi.buduanwang.vip/linux/docker/323.html/)
> - [LskyPro2安装教程 ](https://goojoe.cc/115.html)
> - [开源的兰空图床LskyPro](https://blog.csdn.net/wbsu2004/article/details/118555096)
> - [【教程贴】Docker部署教程和相关问题讨论 · Discussion #373 · lsky-org/lsky-pro (github.com)](https://github.com/lsky-org/lsky-pro/discussions/373)
> - [hellodk34/picgo-plugin-lankong: A PicGo uploader for 兰空图床 lsky-pro，支持 V1 和 V2。 (github.com)](https://github.com/hellodk34/picgo-plugin-lankong)

> 网上好多通过PHP部署或者宝塔部署的，看的头疼，毕竟PHP用的是越来越少了，Docker部署应用方式比较方便，所以PHP的搭建，网上自寻教程。
>
> 另外：宝塔，有毒..................................一点不好用。

## 1.1 Lsky Pro

官网地址：[https://www.lsky.pro](https://www.lsky.pro/)

Lsky Pro 是一个用于在线上传、管理图片的图床程序，中文名「兰空图床」，你可以将它作为自己的云上相册，亦可以当作你的写作贴图库。

兰空图床的上传支持多图上传、队列排队、拖拽上传、预览图片、粘贴上传、快捷复制链接等特点，页面简洁实用，上传页面还支持切换不同的储存，同时管理员还可以对游客、用户进行精确控制上传的图片大小、类型、上传频率、图片审核等限制。

兰空图床支持对已经上传的图片进行管理，支持通过相册对图片进行分类，支持对图片进行别名。不用担心列表因为展示过多的图片消耗太多流量，您在列表所看到的图片，都是经过程序压缩后生成的缩略图，并储存在服务器本地。

图片管理还可以通过鼠标拖拽多选的方式对多张图片进行批量操作，瀑布流布局，以画廊的样式展示，看起来更舒适，并且支持通过鼠标右键进行操作。

兰空图床支持多用户，并可以对其分配不同的角色和储存方案，也可以控制每个用户的最大储存大小、封禁某个用户。用户还支持将自己的图片分享至画廊，让其他用户看到。

在后台管理控制台中，可以很清晰的观察到系统的数据指标，以及储存的占用情况。

## 1.2 Docker启动

简简单单，一下就启动好了，它不香吗？

搞个宝塔差点吐了。

```bash
docker run -d --name lsky -p 8090:80 -v /var/project/lsky:/var/www/html halcyonazure/lsky-pro-docker:latest
```

## 1.3 Nginx反向代理设置

如果使用Nginx反代并设置了HTTPS的话，会出现图片加载错误的情况，解决办法参考 [#317](https://github.com/lsky-org/lsky-pro/issues/317) 在32行添加对应代码后即可成功使用

在Docker内可直接执行以下指令来修改文件内容：

```bash
docker exec -it lskypro sed -i '32 a \\\Illuminate\\Support\\Facades\\URL::forceScheme('"'"'https'"'"');' /var/www/html/app/Providers/AppServiceProvider.php
```

## 1.4 记录Nginx配置

443端口是个人博客网站的，8443是配置LskyPro图床的。

这样，访问你的域名  域名:8443就可以访问了。

```
user  root;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    client_max_body_size 10M;
    sendfile        on;
    keepalive_timeout  65;

    server {
       listen       80;
       server_name  XXXXXXXXXXX.com;
      rewrite ^(.*)$ https://$host$1 permanent;
    }
	
    server {
        listen 8443 ssl;
        server_name XXXXXXXXXXX.com;
        ssl_certificate XXXXXXXXXXX.pem; 
        ssl_certificate_key XXXXXXXXXXX.key; 
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 30m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers HIGH:!aNULL:!MD5:!EXPORT56:!EXP;
        ssl_prefer_server_ciphers on;
        proxy_connect_timeout 500;
        proxy_send_timeout 500;
        proxy_read_timeout 500;
        client_max_body_size 10m;

        location / {
            proxy_pass http://127.0.0.1:8090;
            # 此处后面加了端口号，因为此处，排查了两天问题
            proxy_set_header Host $host:8443;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header REMOTE-HOST $remote_addr;
        }
    }

}
```

## 1.5 配置储存策略

登录进去Lsky Pro后，需要设置储存策略，改为域名的形式。

## 1.6 配置PicGo

- [hellodk34/picgo-plugin-lankong: A PicGo uploader for 兰空图床 lsky-pro，支持 V1 和 V2。](https://github.com/hellodk34/picgo-plugin-lankong)

参考链接配置好PicGo即可。

下载插件lankong。

注意：Version版本要配置对。

Strategy ID，如果你是默认策略，就不用改。

此处，我踩雷，我新建了策略，结果没注意，图片上传上去一直是ip:port的这种形式。

### 1.6.1 补充说明 token 的获取方式

1. 兰空图床 V1 token 的获取方式很简单，注册后进入个人设置页面就能看到，复制后使用即可

2. 兰空图床 V2 token 的获取方式：

   1. 登录到一个 V2 版本的兰空图床，比如 [https://dogimg.com](https://dogimg.com/) 进入 API 接口页面 https://dogimg.com/api 查看获取一个 token 的方式

   2. 使用 postman 之类的 api 调试工具发起一个 http post 请求即可生成一个 token，请求时的细节如下

      1. 请求 url: `https://dogimg.com/api/v1/tokens`

      2. 请求方法: POST

      3. 设置请求头 `Accept` 的值为 `application/json`

      4. 请求体中使用 json 语法填入邮箱和密码

         ```
         {
           "email": "your_username@example.com",
           "password":"your_password"
         }
         ```

      5. 请求成功后得到返回信息中的 `token`，使用 `Bearer `拼接拿到的 token 作为 `Auth token` 填入 PicGo 设置中。注意 Bearer 和 返回的 token 之间有个空格，请严格按照格式填写，程序不会校验这个细节

# 2 Mysql数据库

```bash
docker run -p 3308:3306 --name gly-test -v /home/logs/mysql/test/conf:/etc/mysql/conf.d -v /home/logs/mysql/test/logs:/logs -v /home/logs/mysql/test/data:/var/lib/mysql -v /etc/localtime:/etc/localtime -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7 --lower_case_table_names=1
```

# 3 redis

```bash
docker run --name redis -p 6379:6379 -v /var/project/redis/data:/data -v /var/project/redis/conf:/usr/local/etc/redis/redis.conf -d redis redis-server --appendonly yes

# 命令说明
--name redis         　　　　　　　　 　   启动后容器名为 my-redis
-p 6379:6379           　　　　　　　　　　  将容器的 6379 端口映射到宿主机的 6379 端口
-v /usr/local/workspace/redis/data:/data           将容器  /usr/local/workspace/redis/data 日志目录挂载到宿主机的 /data
-v /home/redis/conf:/usr/local/etc/redis/redis.conf    将容器  /usr/local/etc/redis/redis.conf 配置目录挂载到宿主机的 /conf
redis-server --appendonly yes　　　          在容器执行redis-server启动命令，并打开redis持久化配置
redis-server --requirepass 123456              配置redis的登录密码
```

# 4 监控平台

## 4.1 Prometheus

```bash
docker run -d --restart=always --name=prometheus -p 9090:9090 -v /etc/localtime:/etc/localtime:ro  -v /var/project/prometheus:/etc/prometheus prom/prometheus
```

1. 编写prometheus.yml配置文件

```yaml
# 全局配置
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: 'prometheus监控'  
    static_configs:
    - targets: ['127.0.0.1:9090']
```

## 4.2 Grafana

Grafana模板地址：https://grafana.com/grafana/dashboards

```bash
docker run -d --restart=always --name=grafana -p 3000:3000 grafana/grafana
```

## 4.3 Node exporter

```bash
docker run -d -p 9101:9100 --net="bridge" --pid="host" --name=node-exporter -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter --path.rootfs /host
```

prometheus配置：

```yaml
# 全局配置
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
  # 通过node_exporter将监控数据传给prometheus，如果要监控多台服务器，只要在每个服务器上安装node_exporter，指定不同多ip地址就好了
  - job_name: '服务器监控'  
    static_configs:
    - targets: ['172.21.0.15:9100']
```

## 4.4 MySQL exporter

> Grafana模板：12826

mysqld exporter的功能是收集mysql服务器的数据, 并向外提供api接口, 用于prometheus主要获取数据。

在被监控端mysql服务器上创建账号用于mysql exporter收集使用

```sql
GRANT REPLICATION CLIENT, PROCESS ON  *.*  to 'exporter'@'%' identified by '123456';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
flush privileges;
```

在被监控端mysql服务器上安装mysql exporter。

```bash
docker run -d  --restart=always  --name mysqld-exporter -p 9104:9104   -e DATA_SOURCE_NAME="user:password@(hostname:port)/"   prom/mysqld-exporter

docker run -d  --restart=always  --name mysqld-exporter -p 9104:9104   -e DATA_SOURCE_NAME="exporter:123456@(192.168.1.82:3306)/"   prom/mysqld-exporter

要查看容器是否报错, 主要是验证exporter与mysql服务端之间正常连接和获取数据; 
docker logs -f mysqld-exporter  看有没有报错
```

prometheus.yml

```yaml
# 全局配置
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
  # 通过node_exporter将监控数据传给prometheus，如果要监控多台服务器，只要在每个服务器上安装node_exporter，指定不同多ip地址就好了
  - job_name: '服务器监控'  
    static_configs:
    - targets: ['172.21.0.15:9100']
  - job_name: 'MySQL数据库监控'  
    static_configs:
    - targets: ['172.21.0.15:9104']
```

Mysql监控告警规则：

```bash
    groups:
    - name: MySQL-rules
      rules:
      - alert: MySQL Status 
        expr: up == 0
        for: 5s 
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: MySQL has stop !!!"
          description: "检测MySQL数据库运行状态"

      - alert: MySQL Slave IO Thread Status
        expr: mysql_slave_status_slave_io_running == 0
        for: 5s 
        labels:
          severity: warning
        annotations: 
          summary: "{{$labels.instance}}: MySQL Slave IO Thread has stop !!!"
          description: "检测MySQL主从IO线程运行状态"

      - alert: MySQL Slave SQL Thread Status 
        expr: mysql_slave_status_slave_sql_running == 0
        for: 5s 
        labels:
          severity: warning
        annotations: 
          summary: "{{$labels.instance}}: MySQL Slave SQL Thread has stop !!!"
          description: "检测MySQL主从SQL线程运行状态"

      - alert: MySQL Slave Delay Status 
        expr: mysql_slave_status_sql_delay == 30
        for: 5s 
        labels:
          severity: warning
        annotations: 
          summary: "{{$labels.instance}}: MySQL Slave Delay has more than 30s !!!"
          description: "检测MySQL主从延时状态"

      - alert: Mysql_Too_Many_Connections
        expr: rate(mysql_global_status_threads_connected[5m]) > 200
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: 连接数过多"
          description: "{{$labels.instance}}: 连接数过多，请处理 ,(current value is: {{ $value }})"  

      - alert: Mysql_Too_Many_slow_queries
        expr: rate(mysql_global_status_slow_queries[5m]) > 3
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: 慢查询有点多，请检查处理"
          description: "{{$labels.instance}}: Mysql slow_queries is more than 3 per second ,(current value is: {{ $value }})"
```

## 4.5 blackbox_exporter

```bash
docker pull prom/blackbox-exporter

docker run -d -p 9115:9115 --name blackbox-exporter  prom/blackbox-exporter
```

blackbox_exporter配置

```yaml
modules:
  http_2xx:  # http 监测模块
    prober: http
    http:
  http_post_2xx: # http post 监测模块
    prober: http
    http:
      method: POST
  tcp_connect: # tcp 监测模块
    prober: tcp
  ping: # icmp 检测模块
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
```

Prometheus配置：

### 4.5.1 ping监测

在内网可以通过ping (icmp)检测服务器的存活，以前面的最基本的module配置为例，在Prometheus的配置文件中配置使用ping module：

```yml
- job_name: 'ping_all'
    scrape_interval: 1m
    metrics_path: /probe
    params:
      module: [ping]
    static_configs:
     - targets:
        - 192.168.1.2
       labels:
         instance: node2
     - targets:
        - 192.168.1.3
       labels:
         instance: node3
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 127.0.0.1:9115 # black_exporter 这里和Prometheus在一台机器上
```

通过配置文件可以很直接的看出Prometheus使用black_exporter作为代理使用black_exporter配置的module检测各个target的状态。 下面是一个http://127.0.0.1:9115/probe?module=ping&target=192.168.1.2返回的是192.168.1.2这个target的metrics。

```yml
#DNS解析时间,单位 s
probe_dns_lookup_time_seconds 0.039431355
#探测从开始到结束的时间,单位 s,请求这个页面响应时间
probe_duration_seconds 0.651619323

probe_failed_due_to_regex 0

#HTTP 内容响应的长度
probe_http_content_length -1
#按照阶段统计每阶段的时间
probe_http_duration_seconds{phase="connect"} 0.050388884   #连接时间
probe_http_duration_seconds{phase="processing"} 0.45868667 #处理请求的时间
probe_http_duration_seconds{phase="resolve"} 0.040037612  #响应时间
probe_http_duration_seconds{phase="tls"} 0.145433254    #校验证书的时间
probe_http_duration_seconds{phase="transfer"} 0.000566269
#重定向的次数
probe_http_redirects 1
#ssl 指示是否将 SSL 用于最终重定向
probe_http_ssl 1
#返回的状态码
probe_http_status_code 200
#未压缩的响应主体长度
probe_http_uncompressed_body_length 40339
#http 协议的版本
probe_http_version 1.1
#使用的 ip 协议的版本号
probe_ip_protocol 4

probe_ssl_earliest_cert_expiry 1.59732e+09
#是否探测成功
probe_success 1
#TLS 的版本号
probe_tls_version_info{version="TLS 1.2"} 1
```

### 4.5.2 http监测-监测网站状态

以前面的最基本的module配置为例，在Prometheus的配置文件中配置使用http_2xx module：

```yml
- job_name: 'http_get_all'  # blackbox_export module
    scrape_interval: 30s
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://frognew.com
        - 172.0.0.1:9090
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115 #blackbox-exporter 所在的机器和端口
```

http检测除了可以探测http服务的存活外，还可以根据指标probe_ssl_earliest_cert_expiry进行ssl证书有效期预警。

### 4.5.3 ICMP监测-监控主机存活状态

```yml
- job_name: node_status
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets: ['10.165.94.31']
        labels:
          instance: node_status
          group: 'node'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 172.19.155.133:9115
```

10.165.94.31是被监控端ip，172.19.155.133是Blackbox_exporter

### 4.5.4 TCP监测-监控主机端口存活状态

- 监听 业务端口地址，用来判断服务是否在线，我觉的和telnet 差不多
- 相关代码块添加到 Prometheus 文件内
- 对应 blackbox.yml文件的 tcp_connect 模块

```yml
- job_name: 'prometheus_port_status'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['172.19.155.133:8765']
        labels:
          instance: 'port_status'
          group: 'tcp'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 172.19.155.133:9115
```

### 4.5.5 SSL 证书过期时间监测

prometheus.yml

```yml
rule_files:
  - ssl_expiry.rules
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - example.com  # Target to probe
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # Blackbox exporter.
```

ssl_expiry.rules

```yaml
groups: 
  - name: ssl_expiry.rules 
    rules: 
      - alert: SSLCertExpiringSoon 
        expr: probe_ssl_earliest_cert_expiry{job="blackbox"} - time() < 86400 * 30 
        for: 10m
```

## 4.6 mongodb-exporter

```bash
docker run -itd --name mongo -p 9106:9104 eses/mongodb_exporter --mongodb.uri=mongodb://10.37.193.58:27017
```

# 5 Portiner

```bash
docker run -d -p 9001:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v /data/portainer/public:/public portainer/portainer
```

# 6 Wizard

```bash
docker run -d --name wizard -e DB_HOST=xxx -e DB_PORT=3308 -e DB_DATABASE=wizard -e DB_USERNAME=root -e DB_PASSWORD=123456  -p 8080:80 -v /wizard:/webroot/storage/app/public mylxsw/wizard
```

Wizard上传附件大小配置文件地址：

```bash
/usr/local/etc/php/conf.d/upload-limit.ini
upload_max_filesize = 100M
post_max_size = 0
```

# 7 SkyWalking

- [基于Docker部署Skywalking](https://juejin.cn/post/7086673827542859813)
- [使用skywalking监控springboot项目](https://www.cnblogs.com/Vincent-yuan/p/15764981.html)

## 7.1 ES

```bash
docker run --restart=always -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
--name='elasticsearch' --cpuset-cpus="1" -m 2G -d elasticsearch:7.6.2
```

验证es安装成功

  浏览器地址栏输入：[http://127.0.0.1:9200/，浏览器页面显示如下内容：](https://link.juejin.cn?target=http%3A%2F%2F127.0.0.1%3A9200%2F%EF%BC%8C%E6%B5%8F%E8%A7%88%E5%99%A8%E9%A1%B5%E9%9D%A2%E6%98%BE%E7%A4%BA%E5%A6%82%E4%B8%8B%E5%86%85%E5%AE%B9%EF%BC%9A)

```json
{
  "name" : "6eebe74f081b",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "jgCr_SQbQXiimyAyOEqk9g",
  "version" : {
    "number" : "7.6.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ef48eb35cf30adf4db14086e8aabd07ef6fb113f",
    "build_date" : "2020-03-26T06:34:37.794943Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## 7.2 Skywalking OAP

启动Skywalking OAP

注：--link后面的第一个参数和elasticsearch容器名一致； -e SW_STORAGE_ES_CLUSTER_NODES：es7也可改为你es服务器部署的Ip地址，即ip:9200

```shell
docker run --name oap --restart always -d --restart=always -e TZ=Asia/Shanghai -p 12800:12800 -p 11800:11800 --link elasticsearch:elasticsearch -e SW_STORAGE=elasticsearch7 -e SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200 apache/skywalking-oap-server:8.3.0-es7
```

## 7.3 启动Skywalking UI

注：--link后面的第一个参数和skywalking OAP容器名一致；

```shell
docker run -d --name skywalking-ui --restart=always -e TZ=Asia/Shanghai -p 8088:8080 --link oap:oap -e SW_OAP_ADDRESS=oap:12800 apache/skywalking-ui:8.3.0
```

## 7.4 javaagent

官网下载skywalking-agent，  下载地址：[archive.apache.org/dist/skywal…](https://link.juejin.cn/?target=https%3A%2F%2Farchive.apache.org%2Fdist%2Fskywalking%2F8.3.0%2F)

一定要下载对应的skywalking-oap版本的skywalking-agent，否则就有可能会导致agent无法上报，笔者一开始觉得agent可以做到向下兼容，下载了8.8版本，导致上传监控数据失败了，报错原因是oap-server端返回的方法不存在`grpc-message: Method not found: skywalking.v3.JVMMetricReportService/collect`

```bash
java -jar -javaagent:/Users/penghuiliu/geek_learn/skywalking-agent/skywalking-agent.jar=agent.service_name=fw-gateway,collector.backend_service=127.0.0.1:11800 -jar spring-transaction-2.2.6.RELEASE.jar
```

# 附录A

## A.1 查看运行容器的docker run启动参数

- [查看运行容器的docker run启动参数](https://www.cnblogs.com/aresxin/p/docker-run.html)

### A.1.1 安装pip

```shell
# yum install -y python-pip
```

### A.1.2 安装runlike

```shell
# pip install runlike
```

### A.1.3 查看docker run参数

发布一个测试容器

```shell
# docker run -d -v /data/nginx:/data/nginx -v /etc/hosts:/etc/hosts -p 8080:80 --name nginx nginx:1.18 
# netstat -lntup | grep 8080
tcp6       0      0 :::8080                 :::*                    LISTEN      5357/docker-proxy
# docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                  NAMES
76a49h8f017c   nginx:1.18   "nginx -g 'daemon of…"   2 seconds ago   Up 2 seconds   0.0.0.0:8080->80/tcp   nginx
```

使用runlike查看启动参数

格式：runlike -p <容器名>|<容器ID>

```shell
# runlike -p nginx
docker run \
    --name=nginx \
    --hostname=76a49h8f017c \
    --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    --env=NGINX_VERSION=1.18.3 \
    --env=NJS_VERSION=0.3.9 \
    --env='PKG_RELEASE=1~buster' \
    --volume=/data/nginx:/data/nginx \
    --volume=/etc/hosts:/etc/hosts \
    -p 8080:80 \
    --restart=no \
    --label maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>" \
    --detach=true \
    nginx:1.18 \
    nginx -g 'daemon off;'
```

## A.2 Docker-命令自动补齐

自动补齐需要依赖工具 bash-complete，如果没有，则需要手动安装，命令如下：

`[root@docker ~]# yum -y install bash-completion`

安装成功后，得到文件为  /usr/share/bash-completion/bash_completion。

前面已经安装了 bash_completion，执行如下命令：

`[root@docker ~]# source /usr/share/bash-completion/bash_completion`

再次尝试，发现可以正常列出docker的子命令，示例如下：

```bash
[root@docker ~]# docker  （docker + 空格 + 连续按2次Tab键）
attach    container  engine    history   inspect   logs      port     restart   search    stats    top      volume
build     context    events    image     kill      network   ps       rm        secret    stop     trust    wait
builder   cp         exec      images    load      node      pull     rmi       service   swarm    unpause
commit    create     export    import    login     pause     push     run       stack     system   update
config    diff       help      info      logout    plugin    rename   save      start     tag      version
```
