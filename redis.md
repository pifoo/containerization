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

`cat Dockerfile`

```
FROM ruby:2.3.1-alpine

ADD https://raw.githubusercontent.com/antirez/redis/3.2.0/src/redis-trib.rb /usr/local/bin/redis-trib.rb

RUN gem install redis && chmod 755 /usr/local/bin/redis-trib.rb && \
  sed -i '/yes_or_die.msg/a return if ENV["QUIET_MODE"] == "1"' /usr/local/bin/redis-trib.rb

ADD entrypoint.sh /entrypoint.sh

ENTRYPOINT [“/entrypoint.sh"]

```

脚本文件

`cat entrypoint.sh`

```
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

#### 1.创建模版

![模板](https://github.com/billycyzhang/Shell/blob/master/images/tmp.jpg)

#### 2.添加redis服务-选择镜像

![镜像-1](https://github.com/billycyzhang/Shell/blob/master/images/image-1.jpg)

#### 3.设置容器参数

![容器参数-1](https://github.com/billycyzhang/Shell/blob/master/images/parameter-1.jpg)

#### 4.健康检查

![健康检查-1](https://github.com/billycyzhang/Shell/blob/master/images/check-1.jpg)

#### 5.部署策略

![部署策略-1](https://github.com/billycyzhang/Shell/blob/master/images/d-1.jpg)

###### 添加redis集群初始化服务redis-trib-选择镜像

![镜像-2](https://github.com/billycyzhang/Shell/blob/master/images/image-2.jpg)

###### 1.设置容器参数

![容器参数-2](https://github.com/billycyzhang/Shell/blob/master/images/parameter-2.jpg)

###### 2.部署策略

![部署策略-2](https://github.com/billycyzhang/Shell/blob/master/images/d-2.jpg)

**我们基于刚才的redis-sharding模版,就可以实现一键部署一个redis cluster出来**

![应用实例](https://github.com/billycyzhang/Shell/blob/master/images/app-i.jpg)

查看redis-trib集群初始化后的结果，我们看到集群的初始化过程都很正常。

![初始化结果](https://github.com/billycyzhang/Shell/blob/master/images/i-result.jpg)

登录到任意一台redis节点执行redis-cli info:

![最终结果](https://github.com/billycyzhang/Shell/blob/master/images/result.jpg)

谢谢大家，我今天就分享到这里！

Q1: 如果我想在一个机器上部署多个redis实例可以吗？
A1: 可以

Q2: 问下你们ui编排工具是自主研发还是基于什么开源工具？内部逻辑是什么.?
A2: 自主研发的。通过易用的界面对docker容器运行参数进行设置和保存。每个容器运行参数和优先级以及部署策略构成一个个服务，多个服务组合成一个可以一键部署的应用模版。

Q3：redis集群的docker我看ip都是同网段的。这个是单独的docker管理工具处理的？如果只单纯搭建redis集群，而无docker集群管理。这个多个redis node如何管理？
A3：csphere平台内部支持网络管理功能，并在背后拥有自动的可编程的服务发现能力，使得自动化部署集群成为可能。如果脱离docker手工部署，按照官方文档一步步操作即可

Q4: 请问这里的模板数据是何时传入进去的？
A4: 模板数据分两种：
1，配置文件模板里定义的模板变量，这类数据是在创建应用实例时用户通过cSphere管理平台填写的；
2，集群服务相关的元数据，如每个容器的IP地址、容器所在的主机参数等，这类数据是cSphere应用编排引擎在创建应用实例时，自动从集群各节点收集并注册到配置模板解析引擎的
配置文件模板经解析生成最终配置文件，然后装载到每一个容器里

Q5：请问redis－cluster的扩容、缩容，resharding如何处理的呢？
A5: 扩容增加节点的话，触发trib脚本重新resharding，减少节点的话，则需要在前面先执行，trib脚本里面有添加删除节点的命令

Q6: 这是你们的商业平台？还是openstack集成docker的结果？
A6：我们的商业平台，为企业提供整体的PaaS解决方案。希云cSphere平台底层是docker，希云cSphere平台可以部署在OpenStack平台之上。

Q7: redis3.0目前自己出的这个Q5集群方案稳定吗？有没有经过大量的数据测试！效率如何？因为我不是专业做运维的，我是做开发的对运维的知识比较感兴趣但是不专业，希望能得到一个经过数据支撑的答案 
A7: redis当前的集群稳定性是比较好的，国内外有大量互联网企业大规模的使用，据我所知，唯品会的redis集群规模在500台以上

Q8: 你们的pass平台在部署容器时还能指让用户自主定制部署策略？这样做的目的是什么?
A8：不同类型的应用有不同的资源偏好，比如CPU密集型的，磁盘IO密集型的，通过调度策略的选择，用户可以更深度的控制容器在主机集群上的分布，使应用获得更好的运行效果。

Q9: redis用docker做集群，在内存方面有什么需要额外注意的地方吗？
A9：内存方面注意设置内核vm相关参数，另外配置文件里可以加入内存最大大小的设置等，如果要自动化，可以自动获取容器的内存配额或主机节点的内存size自动计算