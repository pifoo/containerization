> 大家好，我是希云docker工程师张春源，接触docker是在2013年,那会国内还没有docker的创业公司。

分享进入正文：

### docker为什么如此火热？

> 个人觉得，搞技术的最终都是要去解决生活中的问题，光搞技术带不来什么价值。所以我们先来回顾一下：第一艘集装箱船是美国于1957年用一艘货船改装而成的。它的装卸效率比常规杂货船大10倍，停港时间大为缩短，并减少了运输装卸中的货损。从此，集装箱船得到迅速发展。


![集装箱]()

集装箱的优点：

1. 节省劳动力，减少运输费用

2. 减少货物损耗和损坏，保证货物质量

3. 集装箱装卸效率高

4. 规格标准化，易运输

现在集装箱已经为我们的生活带来的很大的变化，集装箱更是运输行业的一次重大**革命**

![docker](https://discuss.csphere.cn/uploads/default/original/2X/c/c8cdc4319950032cc334d1634e582de67cbdb536.jpg)

docker中文意思是码头工人，他的职责应该是管理**集装箱**--容器containers.

所以火热的应该是容器技术，而docker是容器技术的代表。Coreos官方也推出了rkt,来运行容器。

既然是容器技术如此火热，那么容器技术有哪些特点，让企业和IT人员如此热爱？

1.可以提升研发和测试的效率，生成运行环境更容易

2.更容易实现devopts

3.研发以镜像作为交付，让环境一致化

4.开发，测试，运维协作更容易

5.一次构建（build），多处运行

...等等优势

### 什么是docker，给企业带来哪些价值？

以上的优势都是由·linux内核和cgroup带来的。docker是基于linux内核的虚拟化技术，所以对系统的内核版本有要求，是因为低版本的内核不支持namespace，因为共用宿主机内核，所以容器container的性能接近于native的性能。通过5大namesapce实现了运行环境的隔离(第6个user namespace现在已在试验版本中)，又通过cgroup实现了资源限制。

举个例子来说一下用docker后的工作流程应该是：

需求：开发一个php的wordpress博客。

1.运维和开发协同写Dockerfile,生成php的的镜像

2.开发拿到php的镜像，拉倒本地后启动php容器，进行开发。

3.开发完毕，研发把代码和php的镜像结合，打包成一个应用镜像（app image）,交付到测试环境

4.测试人员得到通知，那到app image 创建测试环境，不通过研发继续改，通过后给镜像打个状态(testing pass)

5.运维人员拿到镜像，传入不同的运行参数，上线，上线，上线！

注：java项目类似。

下图是6个namespace对linux内核版本的支持列表：

![namespace](https://discuss.csphere.cn/uploads/default/optimized/2X/1/17c62bd4ef148704072466f54b20dbc54f83c7fa_1_690x451.jpg)

### docker架构

docker就是C/S架构

1.在linux系统上安装docker，所输入的类似`docker images`、 `docker ps`等命令，都是docker client端。

2.执行命令后都是由docker daemon来进行处理，这就是docker server端

对docker的架构非常好理解，运维人员对docker架构熟悉即可。

### Dockerfile,image,container的关系

用一张图来描述它们之间的关系：

![Dockerfile](https://discuss.csphere.cn/uploads/default/original/2X/e/e1f44504650fd44fe781ffe017a497d2c8e8cd17.jpg)

看图理解它们之间的关系，不再过多描述

### docker存储

docker的存储包括2个方面：

1.容器和镜像存储


2.文件数据存储

---

容器和镜像的存储也就是docker所支持的storage driver：

- AUFS

- Overlay

- Devicemapper

分别来说一下，这3种存储的特点：

在了解存储特点的时候，我们需要了解一下docker镜像，docker镜像是通过写时复写机制实现了分层，也就是说一个镜像是都多个层（layer）构成。

Aufs是Another Union File System的缩写，支持将多个目录挂载到同一个虚拟目录下。

如下图：
![aufs](https://discuss.csphere.cn/uploads/default/original/2X/7/76906cc145e6e709291136d1313dd864c3609a95.jpg)

已构建的镜像会设置成只读模式，read-write写操作是在read-only上的一种增量操作，固不影响read-only层。

举例：现在构建一个php的镜像，如果想基于php镜像构建app镜像，只需要继承(from)php镜像即可，而app镜像不影响php的镜像，只是在php的基础上继续叠加层。

Overlay是一个叠加文件系统，用于叠合多个文件系统形成一个新的文件系统,使用方式如下：

`mount -t overlay overlay -olowerdir=/lower, upperdir=/upper, workdir=/work /merged`

overlayfs通过Linux内核VFS层的灵活性能够将对文件A的修改变成对B的修改，利用这种灵活性来完成文件系统叠加的效果!

Overlay只有2层：

- lower：通常是只读的，镜像文件存储到lower层，如果想对lower层的文件做修改，需要把lower层的文件copy到upper才能更新，这也就是实现了CoW机制，但如果lower层有一个非常大的文件需要进行修改，这样的场景下就不推荐使用overlay。不过overlay需要3.18的内核才行。

Devicemapper是在linux kernel2.6提供的一种从逻辑设备到物理设备的映射框架机制。

devicemapper的精简配置模块实现了镜像的分层，这个模块使用了2个设备，1个用于存储元数据（metadata），1个用于存储数据(data)。data就是一个资源池，为其他块设备（容器）提供资源；metadata存储了虚拟设备和物理设备的映射关系。

CoW机制是在块存储级别发生，通过对已有设备创建快照的方式创建新的设备，这些新的快设备在写入内容之前并不会分配资源。docker使用devicemapper存储驱动，所有的容器和镜像都有自己的块设备，所以在任何时候都能创建快照提供新的容器和镜像。

devicemapper是基于块粒度的CoW,这个粒度要小于overlay的，并且如果底层有一个文件需要修改，devicemapper只需要copy那个需要更改的块就可以了，无需全部copy。

> 以上内容已明白，Docker镜像是由一系列的只读层叠加而成，当启动一个容器时，Docker加载镜像的所有只读层，并在最上面加入一个读写层，这样就形成了我们所说的容器。

分层的实现方式，可以提高镜像的构建，存储和分发并可以节省存储空间，但是还是不存在一些问题不能满足，比如：

- 容器中的数据如果想进行保存和修改，在宿主机上无法操作

- 容器之间的数据无法共享

- 容器中的数据会随着容器被删除而删除掉

那么针对以上这些问题，出现了`docker volume`

2.文件数据存储VFS，vfs不支持写时复写。

docker volume是独立于联合文件系统，通过VFS文件系统而实现。

- volume可以在容器间进行数据共享和重用

- 对volume目录下的文件进行修改，可以在容器中马上生效

- 删除容器后不删除volume，即使没用容器在使用volume也不会被删除

- 可以重复挂载

提示：运行DB类的容器，需要把数据库目录下文件通过docker -v挂载到宿主机目录下。

总结：

如果我们想利用docker，那么我们需要付出哪些代价：

- 运维需要熟悉使用docker

- 运维需要写Dockerfile

- 研发协同运维编写Dockerfile生成镜像

- 测试，了解docker即可，无需深入

- 研发在项目代码中放一个Dockerfile和.dockerignore文件

另外，应用docker目前推荐跑web类应用，推荐是微服务架构的业务。

今天的分享到此结束，欢迎提问。
