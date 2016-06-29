# Centos7 基础镜像

## 基于官方centos:7.2.1511

`docker pull centos:7.2.1511`

## 设置系统时区

`ENV TZ "Asia/Shanghai"`

## 添加基础软件包

```
RUN yum -y install curl wget tar bzip2 unzip vim-enhanced passwd sudo yum-utils hostname net-tools rsync man && \
    yum install -y gcc gcc-c++ git make automake cmake patch python-devel libpng-devel libjpeg-devel && \
    yum install -y --enablerepo=epel pwgen python-pip && \
    yum clean all
```

## 安装 supervisor进程管理器

```
RUN pip install supervisor && \
    mkdir -p /etc/supervisor.conf.d && \
    mkdir -p /var/log/supervisor
```

*注：`/etc/supervisor.conf.d`此目录下，在supervisor启动时会把响应的服务拉起；`/var/log/supervisor` supervisor日志存储目录；

## 添加supervisor主配置文件

`COPY supervisord.conf /etc/supervisord.conf`

## 启动supervisor服务，且PID为1

`CMD ["/usr/bin/supervisor", "-n", "-c", "/etc/supervisord.conf"]`

*注：`-c,--configuration`参数：加载主配置文件；

> 联系邮箱: “zcy@nicescale.com”  weixi: "kernel0963"