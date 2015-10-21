# 如何部署企业内部Registry仓库（带SSL证书）

> 企业使用docker技术以后，不管是快速生成环境还是交付产品，效率都是得到了很大的提升。镜像仓库（Registry）在传输镜像的过程中起非常重要的作用，那么企业构建自己内部的私有Registry是必须执行的任务。


部署Registry镜像仓库非常简单，如果要是配置用户认证和添加SSL证书，网上查到的解决办法大多都需要用到nginx，对于Registry V2版本简化了部署。 

**V2对于V1的一些变化：**

1.Go替换Python

2.镜像下载上传效率提升

3.内嵌webhook

4.部署简化

> Registry镜像的**volume目录**已发生变化：`/tmp`-->`/var/lib/registry`

不过对于Registry真是醉了，旧的问题解决了，又引来了新的问题！不要问我新问题是神马，你要自己去发现！

## 构建私有Registry

**下载官方Registry镜像**

`docker pull registry:2.1.1`

## 生成证书

**通过openssl命令生产证书，比如域名是reg.domain.com**

`mkdir certs`

`openssl req -x509 -days 3650 -subj '/CN=reg.domain.com/' -nodes -newkey rsa:2048 -keyout registry.key -out registry.crt`

## 生成密码

**从本地push镜像到官方Hub，需要用户认证。实现从本地到私有Hub push也需要用户认证**

`mkdir auth`

`docker run --entrypoint htpasswd registry:2 -Bbn testuser password > auth/htpasswd`

> `$testuser` `$password` 随便设置 

## 启动Registry容器

```
docker run -d -p 5000:5000 --restart=always --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  registry:2.1.1
```

> 检查Registry容器是`UP`状态，`docker ps -a`



## 配置本机Registry证书

`mkdir -p /etc/docker/certs.d/reg.domain.com:5000`

`cp /certs/registry.crt /etc/docker/certs.d/reg.domain.com:5000`

## 配置其他主机Registry证书

example：

Registry启动在A主机：192.168.1.100

B主机pull Registry镜像，在B主机上操作：

添加`/etc/hosts/`

`192.168.1.100 reg.domain.com`

注：已作DNS无需此操作！

`scp -r root@192.168.1.1:/etc/docker/certs.d/reg.domain.com:5000 .`

## 测试pull,push

A,B主机执行命令一样：

1.`docker login reg.domain.com:5000`

2.`docker tag busybox reg.domain.com:5000/busybox`

3.`docker push reg.domain.com:5000/busybox`


