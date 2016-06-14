大家好，我叫张春源，希云cSphere合伙人，现在公司主要是做企业级PaaS产品，致力于把容器技术真正落地到企业当中。非常感谢肖总提供这么好的技术交流平台让我们大家有机会一起交流实践容器的经验。

我今天分享的主题是：如何利用容器实现生产级别的Redis sharding集群的一键交付

Redis在很多公司中已经得到了大规模使用，今天我主要介绍Redis sharding跑在容器中的实践。

## 什么是Redis sharding集群
Redis(redis.io)作为最流行的KV数据库，很长一段时间都是单机运行，关于如何实现Redis的数据在多个节点上的分布，在Redis3.0出来之前，有很多第三方的方案。

建议大家参考这个链接。http://redis.io/topics/partitioning

#### Client hash
这是最简单的实现，通过在客户端利用一致性hash算法，将数据分布到不同节点。这种方法的缺点非常明显，缺少故障自动failover能力，并且在扩容时数据分布的搬迁，也比较费劲。

#### 代理模式
一个是Redis官方推荐的Twemproxy，是由twitter公司开发；另一个是国内豌豆荚开源的codis；代理模式最大的好处是仍然使用redis单机的sdk进行开发，维护简单。

#### Redis Cluster
redis3.0继2.8推出sentinel主从自动failover功能后，推出了sharding集群，这就是Redis Cluster，本次分享主要是介绍如何将Redis集群实现一键的部署。
参考文档 http://redis.io/topics/cluster-tutorial

## 首先准备redis镜像
Redis官方已经提供了Redis 3.2和3.3的镜像，都可以用来作为Redis集群的镜像，3.2是稳定版本。目前官方推出了alpine版本的Redis镜像，alpine镜像的优势是体积小。此次分享是采用官方的redis:3.2-alpine的镜像来做集群。

## 准备初始化脚本的执行环境
redis官方提供了一个ruby的脚本redis-trib.rb，这个脚本可以用来初始化集群、resharding集群、rebalance集群等，我们使用官方的脚本来初始化集群。该脚本的运行需要ruby环境，我们来构建一个redis-trib镜像：

以下是构建redis-trib镜像的Dockerfile内容：

# cat Dockerfile

```
FROM ruby:2.3.1-alpine

ADD https://raw.githubusercontent.com/antirez/redis/3.2.0/src/redis-trib.rb /usr/local/bin/redis-trib.rb

RUN gem install redis && chmod 755 /usr/local/bin/redis-trib.rb && \
  sed -i '/yes_or_die.msg/a return if ENV["QUIET_MODE"] == "1"' /usr/local/bin/redis-trib.rb

ADD entrypoint.sh /entrypoint.sh

ENTRYPOINT [“/entrypoint.sh"]

脚本文件

# cat entrypoint.sh

#!/bin/sh

if [ "$CLUSTER_CMD" = create ]; then
  if [ -f /usr/local/etc/redis-trib.conf ] ; then
    . /usr/local/etc/redis-trib.conf
    QUIET_MODE=1 redis-trib.rb create --replicas $REPLICAS $NODES
  fi
fi
```

上面两个文件用来构建redis-trib镜像，Dockerfile中的逻辑比较简单，将github中的redis-trib.rb文件添加到镜像中，并让脚本执行支持非交互模式(QUIET_MODE)。镜像启动时，将执行集群初始化命令。

## 准备配置文件

#### 准备redis集群配置文件

```
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

redis集群的配置文件我们一般放到数据目录/data下，redis进程对/data目录拥有可读写的权限。

#### 准备redis-trib脚本配置文件，用于集群初始化参数获取

entrypoint.sh文件中，最主要的是读取redis-trib.conf配置文件，配置文件的格式非常简单

```
REPLICAS={{.REPLICAS_NUM}}
{{ $rs := service "redis" }}
NODES="{{range $i,$rc := $rs.Containers}} {{$rc.IPAddr}}:6379{{end}}"
```

REPLICAS的意思是每个分片有几个slave，一般配置1个slave ,即REPLICAS=1
NODES的意思是集群的每个节点，包括master和slave。所以如果有10个节点，REPLICAS=1的话，那么将有5个分片(slices)。

## 编排集群
准备好上述镜像和配置文件后，我们开始编排集群

#### 创建模版

![https://github.com/billycyzhang/Shell/blob/master/images/%E6%A8%A1%E6%9D%BF.jpg](模板)

#### 添加redis服务-选择镜像

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F.jpg](镜像-1)

#### 设置容器参数

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%AE%B9%E5%99%A8%E5%8F%82%E6%95%B0-1.jpg](容器参数-1)

#### 健康检查

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%81%A5%E5%BA%B7%E6%A3%80%E6%9F%A5-1.jpg](健康检查-1)

#### 部署策略

![https://github.com/billycyzhang/Shell/blob/master/images/%E9%83%A8%E7%BD%B2%E7%AD%96%E7%95%A5-1.jpg](部署策略-1)

###### 添加redis集群初始化服务redis-trib-选择镜像

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F-2.jpg](镜像-2)

###### 设置容器参数

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%AE%B9%E5%99%A8%E5%8F%82%E6%95%B0-2.jpg](容器参数-2)

###### 部署策略

![https://github.com/billycyzhang/Shell/blob/master/images/%E9%83%A8%E7%BD%B2%E7%AD%96%E7%95%A5-2.jpg](部署策略-2)

**我们基于刚才的redis-sharding模版,就可以实现一键部署一个redis cluster出来**

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%BA%94%E7%94%A8%E5%AE%9E%E4%BE%8B.jpg](应用实例)

查看redis-trib集群初始化后的结果，我们看到集群的初始化过程都很正常。

![https://github.com/billycyzhang/Shell/blob/master/images/%E5%88%9D%E5%A7%8B%E5%8C%96%E7%BB%93%E6%9E%9C.jpg](初始化结果)

登录到任意一台redis节点执行redis-cli info:

![https://github.com/billycyzhang/Shell/blob/master/images/%E7%BB%93%E6%9E%9C.jpg](最终结果)

谢谢大家，我今天就分享到这里！