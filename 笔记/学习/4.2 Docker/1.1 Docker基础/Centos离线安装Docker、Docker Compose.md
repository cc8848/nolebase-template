- [Centos7离线安装Docker](https://blog.csdn.net/yinjl123456/article/details/116799121)

# 一、离线安装Docker

## 1.1 下载Docker并安装

[docker包下载地址](https://download.docker.com/linux/static/stable/x86_64/)

```shell
#下载docker-20.10.0包
https://download.docker.com/linux/static/stable/x86_64/docker-20.10.0.tgz

#上传到Centos系统/data/目录,如
scp docker-20.10.0.tgz root@192.168.0.5:/data/

#进入data目录,解压docker包
cd /data
tar -zxvf docker-20.10.0.tgz

#将解压出来的docker文件内容移动到 /usr/bin/ 目录下
cp docker/* /usr/bin/

#查看docker版本
docker version

#查看docker信息
docker info
```

## 1.2 配置Docker开机自启动服务

```shell
#添加docker.service文件
vi /etc/systemd/system/docker.service
#按i插入模式,复制如下内容:
```

```bash
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
  
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
  
[Install]
WantedBy=multi-user.target
```

```bash
#添加文件可执行权限
chmod +x /etc/systemd/system/docker.service

#重新加载配置文件
systemctl daemon-reload

#启动Docker
systemctl start docker

#查看docker启动状态
systemctl status docker

#查看启动容器
docker ps

#设置开机自启动
systemctl enable docker.service

#查看docker开机启动状态 enabled:开启, disabled:关闭
systemctl is-enabled docker.service
```

# 二、离线安装Docker Compose

下载地址：https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-linux-x86_64

上传至服务器，修改执行权限，存放路径是/usr/local/bin/docker-compose

```bash
chmod +x docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
docker-compose -v
```
