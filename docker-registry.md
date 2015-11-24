
> 很多人问我，虚拟机镜像和docker镜像的区别是什么？其实区别非常明显，我们可以通过阅读Dockerfile文件就可以知道这个镜像都做了哪些操作，能提供什么服务；但通过虚拟机镜像，你能一眼看出来虚拟机镜像里面多做了哪些操作，能提供什么服务吗？更突出的是我们都说是mysql镜像，Wordpress镜像，从不说是虚拟机镜像。这点就更能说明docker是更贴近应用的，不单单是解决底层运行环境。


那么有了docker又如何呢？

1.我们可以快速构建,整体打包 (Build)，在这个过程中主要是运维和研发需要更多的协助，编写Dockerfile来构建镜像
2.快速交付（Ship）,研发以镜像作为交付，整体交付给测试人员，缩短研发和测试之间处理问题的周期
3.快速部署（Run）,docker镜像保证了环境的一致性，让问题在部署前得到解决，更重要的是可以基于镜像，快速部署应用

总结起来其实所有的工作流程，都是围绕docker镜像来完成的。docker镜像贯串整个工作流程，那么镜像构建，交付，运行，以及镜像存储都非常重要。

当公司开始使用docker，到官方的docker hub上下载（pull）镜像，显然很不切合实际，而且咱这公司的网络环境是私有环境，都不允许访问外网，那就更不可能到官方的hub去下载镜像。所以我们接下来分享**实战构建企业级的Docker Registry Server**

然后我们来理解一下什么是Docker镜像，什么是dockerregistry。理解docker镜像和docker registry的关系也非常容易。我们把docker镜像看成是“源代码文件“，registry server就是”git仓库“，平日我们写好的代码文件都需要push到代码仓库中，对于docker镜像也一样，镜像打包好以后需要提交到registry server服务器上让测试人员构建测试环境，或者是上线业务。

公司业务不仅仅是单个，而且还会越来越多，那么镜像也就相对会越来越多，我们需要重点考虑如何管理镜像之间的依赖关系，并要实现自动构建，实现持续集成。

我们需要考虑镜像存储到什么地方，确保镜像安全可用。

1.我们可以把镜像存储到宿主机本地，但这样传输镜像不方便，可靠性低，而且镜像唯一性不好确定。

2.所以我们需要把镜像统一都存储到registry server上，这样传输镜像就更加方便，也能保证镜像的唯一性，并且可靠性高，registry server可以把镜像存储到存储服务器上。

目前支持的后端存储有，openstack-swift,S3,Azure,OSS(aliyun),GCS(Google cloud storage),今天实战文章不去实战此过程。

> 镜像仓库好比就是APP store，我们可以到仓库里面去挑选自己想要的APP,然后下载到手机或者电脑上进行安装使用。docker镜像和仓库也类似。

目前docker registry版本是2.2，也是当前最新的版本。

registry2.2的特性有，目前是用go语言去写的，性能提升比v1能高2-3倍，安全和性能上有很多的提升，那么v1有哪些安全隐患呢？

v1版本，镜像的id是随机生成的，所以每次构建一个层都会随机生成一个新ID，即使是层的内容相同。这样会有一个风险就是层的内容文件会被串改，因为最终验证的是id，而不是里面的内容。

v2版本，镜像ID是通过sha256做hash得出来的，这样一来同样的内容就会得到的是一样的ID。镜像id这点能保证了，但还是有其他的问题。细心的同学会发现运行`docker pull`镜像下载完后，会看到`Digest`字段，看起来docker像是想用此字符来取代`tag`.只是猜测不知道后续会发展成什么一样。

> 这个地方普及一个知识点，有人问，我在dockerhub上能看到镜像的说明文档（README.md），而在docker registry上什么也看不到。主要是因为docker hub上默认是调用github上的秒速文件，所以我们可以想看代码的README文件一样去了解镜像。我们也不需要去花费力气去解决这个问题，但我们也必须应该要把Dockerfile放到git上或者是svn上，让docker镜像真正的代码化。这一点做到后，后面能省非常多的力气，而且这样也非常容易实现CI过程。
 
提示，在实际生产环境中不要使用latest作为镜像的`tag`,推荐在测试过程中镜像以`commit id`作为镜像的`tag`，到生产环境的镜像以产品stalbe版本号作为`tag`。

我们来回顾一下registry v1版本我们是如何实现push镜像的，要么需要配置--insecure-registr=0.0.0.0/0,要么需要配置一个nginx来实现用户验证和配置证书。

这篇文章主要可以概括为：真实经验交流，并实战构建docker registry。

下面命令其实也比较简单，大家看的不要烦啊！现在我们来构建一个有证书，有用户验证的registry server。

## 创建registry server端

1.下载registry2.2镜像

`docker pull registry:2.2`

2.生成自签名证书，如果是购买的证书就不用了，直接用购买的证书即可。假如域名是：`reg.carson.com`

创建目录：

`mkdir registry && cd registry && mkdir certs && cd certs`

`openssl req -x509 -days 3650 -subj '/CN=reg.carson.com/' -nodes -newkey rsa:2048 -keyout registry.key -out registry.crt`

3.生成用户和密码

`cd .. && mkdir auth`

`docker run --entrypoint htpasswd registry:2.2 -Bbn testuser password > auth/htpasswd`

> 用户：testuser  密码：password 可随便填写自己想填写的

4.启动registry server

`docker run -d –p 5000:5000 --restart=always --name registry -v `pwd`/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v `pwd`/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key -v /data:/var/lib/registry registry:2.2 `

> 确认registry server是`UP`状态，`docker ps -a | grep registry`

## 配置docker client端

同registry server在同一台服务器上配置：

1.创建证书目录(没有此目录自己创建，注意端口号)

`mkdir -p /etc/docker/certs.d/reg.carson.com:5000`

2.下载证书

`cp /certs/registry.crt /etc/docker/certs.d/reg.carson.com:5000`

3.域名解析,如果有DNS解析无需做此步骤（registry-server-ip=192.168.1.10）

`echo 192.168.1.10 reg.carson.com >> /etc/hosts`

其他主机配置：

1.创建证书目录(没有此目录自己创建，注意端口号)

`mkdir -p /etc/docker/certs.d/reg.carson.com:5000`

2.下载证书

`scp -r root@192.168.1.10:~/registry/certs/registry.crt /etc/docker/certs.d/reg.carson.com:5000`

3.域名解析,如果有DNS解析无需做此步骤（registry-server-ip=192.168.1.10）

`echo 192.168.1.10 reg.carson.com >> /etc/hosts`

## 验证测试

1.登陆(注意加端口号)

`docker login reg.carson.com:5000`

2.输入用户testuser，密码password以及邮箱

3.更改镜像`tag`

`docker tag busybox reg.carson.com:5000/busybox:1.0`

4.`push`镜像

`docker push reg.carson.com:5000/busybox:1.0`