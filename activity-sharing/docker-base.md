# 课程所用到的命令

## 一、Docker 基础

安装curl命令

`which curl`

`apt-get install -y curl`

`yum -y install curl`

检查内核版本

`uname -r`

安装Docker

`curl -sSL https://get.docker.io/ | sh`

## 二、下载镜像

查看Docker命令帮助：

`docker —help`

下载`centos:7.2.1511`镜像

`docker pull centos:7.2.1511`

官方仓库地址：`hub.docker.com`

启动第一个容器

`docker pull hello-world`

`docker run -it --rm hello-world`

命令介绍

`start`,`stop`,`restart`,`exec`,`rename`,`update`,`logs`

## 三、Dockerfile介绍

oschina [Dockerfile详细介绍](http://t.cn/RU7Jlbf)

github [Dockerfile详细介绍](https://github.com/billycyzhang/Shell/blob/master/Dockerfile%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3.md)


## 四、构建镜像

`clone`代码

`git clone https://github.com/billycyzhang/odt.git`

构建base基础镜像：

```
cd centos7
docker build -t zcy/centos7 .
```

构建nginx+php-fpm镜像

```
cd php-fpm
docker build -t zcy/php-fpm .
```

构建Mariadb镜像

```
cd mariadb
docker build -t zcy/mariadb .
```

创建mariadb容器

```
docker run -d -p 3306:3306 --name mydb -e DB_USER=myuser -e DB_PASS=mypass -e REMOTE_ADMIN=true -e REMOTE_ADMIN_USER=zcy -e REMOTE_ADMIN_PASS=zcypaas --restart=always -v /db-data:/var/lib/mysql zcy/mariadb
```

构建wordpress镜像

```
cd wordpress
docker build -t zcy/wordpress: .
```

创建Wordpress容器

```
docker run -d -p 80:80 --name blog --restart=always -e WORDPRESS_DB_USER=zcy -e WORDPRESS_DB_PASSWORD=mypaas -e WORDPRESS_DB_HOST=172.17.0.1 -v /data/logs:/var/log/nginx zcy/wordpress
```

## 五、Docker Registry管理

`115.182.107.187 reg.zcy.com` >> /etc/hosts

`docker pull reg.zcy.com:8888/registry:2.4.0`
`docker tag reg.zcy.com:8888/registry:2.4.0 registry:2.4.0`

`mkdir registry && cd registry`

生成证书

```
mkdir certs
cd certs
openssl req -x509 -days 3650 -subj '/CN=reg.odt.com/' -nodes -newkey rsa:2048 -keyout domain.key -out domain.crt
```

生成密码

```
mkdir auth
docker run --entrypoint htpasswd registry:2.4.0 -Bbn zcy password > auth/htpasswd
```

创建registry容器

```
docker run -d -p 8888:5000 --restart=always --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -v /registry:/var/lib/registry \
  registry:2.4.0
```
配置Registry证书

`mkdir /etc/docker/certs.d/reg.odt.com:8888`

`cp /root/certs/domain.crt /etc/docker/certs.d/reg.odt.com:8888`

```
docker tag busybox reg.odt.com:8888/busybox
docker login reg.odt.com:8888
docker push reg.odt.com:8888/busybox
```

## 六、日志管理

下载`elk`镜像

`docker pull reg.zcy.com:8888/elk`

启动elk容器

`docker run -d -p 5601:5601 -p 9200:9200 -p 5044:5044 -p 5000:5000 --name elk reg.zcy.com:8888/elk`

> 登陆kibana管理面板: `http://<your-host>:5601`


下载`filebeat`镜像

`docker pull reg.zcy.com:8888/filebeat`

创建filebeat容器

`docker run -d --name filebeat -v /data/logs:/data/logs -v /var/lib/docker/containers:/var/lib/docker/containers --link elk:elk reg.zcy.com:8888/filebeat`

登陆kibana管理面板: `http://<your-host>:5601`,查看日志信息；

`logstash-*` to `filebeat-*`

# 八、网络实战

- 服务发现 etcd、consul、zookeeper

- docker 1.9+

### 更改docker daemon启动参数

```
vim /etc/systemd/system/multi-user.target.wants/docker.service

ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --cluster-store=consul://172.16.0.87:8500 --cluster-advertise=eth0:2376
```

### 下载安装consul

`docker run -d -p 8400:8400 -p 8500:8500 -p 8600:53/udp -h consul reg.zcy.com:8888/consul -server -bootstrap -ui-dir /ui`

115.182.107.187


> 注意ip地址

### 确认docker daemon启动参数更改是否生效

`ps -aux | grep docker`

### 主机-B 上，更改`docker`启动参数：

```
ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --cluster-store=consul://172.16.0.87:8500 --cluster-advertise=eth0:2376
```

## 九、实战Swarm

### 下载镜像

`docker reg.zcy.com:8888/swarm`

### 确保`consul`服务正常运行

`docker ps -a | grep consul`or`登陆控制面板进行查看`

### 安装控制器`manager`

`docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise 172.16.0.87:4000 consul://172.16.0.87:8500`

### 安装节点

`docker run -d swarm join --advertise=172.16.0.87:2376 consul://172.16.0.87:8500`

### host-B主机安装节点(172.16.0.148)

`docker run -d -p 4000:4000 swarm manage -H :4000 --replication  --advertise 172.16.0.148:4000 consul://172.16.0.87:8500`

### 加入集群
`docker run -d swarm join --advertise=172.16.0.148:2376 consul://172.16.0.87:8500`

**构建Swarm Manage HA 按照以上命令执行；**

## 十、持续集成

### 安装部署gitlab

```
docker run --detach \
    --hostname gitlab.example.com \
    --publish 443:443 --publish 80:80 \
    --name gitlab \
    --restart always \
    --volume /srv/gitlab/config:/etc/gitlab \
    --volume /srv/gitlab/logs:/var/log/gitlab \
    --volume /srv/gitlab/data:/var/opt/gitlab \
    reg.zcy.com:8888/gitlab-ce:latest
```

http://<your_ip>

**默认登陆用户密码**

```
`user`:`root`
`password`:`5iveL!fe`
```
### 部署`jenkins`

```
docker run -d -p 8080:8080 -p 50000:50000 -v /var/jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock reg.zcy.com:8888/jenkins:1.625.2
```
### 配置项目

```
REG=192.168.55.2
TAG=`cat version | awk -F= '{print $2}'`
IMAGE=$REG/discuss:$TAG

docker build $IMAGE .
docker push $IMAGE
```