# 5.安装cSphere Agent节点

## 5.1 生成COS验证码

COS验证码是Agent加入管理节点的口令，生成COS验证码操作步骤如下：

1.登录管理节点Web页面

2.点击系统设置

3.点击生成COS验证码

**重要提示**:生成验证码的页面在Agent节点没有安装完成前，不要点击结束安装，也别关闭页面。

## 5.2 安装Agent

*前提*：将csphere-agent的rpm包上传至服务器

## 5.3 初始化Agent

`VMWare`平台网络要选择`IPVlan`模式进行初始化

```
Role=agent ControllerAddr=192.168.122.10:80 InstCode=6906 NetMode=ipvlan InetDev=eth0 csphere_init
```
参数说明：

- `ControllerAddr`: 管理节点的`地址`:`端口`(192.168.1.2:80)
- `InstCode`: 安装码, 生成验证码请按照`5.1步骤`进行操作
- `NetMode`: Docker容器网络模式, 填写ipvlan
- `InetDev`: 物理网卡名，替换成实际网卡的名称

## 5.4 启动Agent

```
# 启动过程中docker启动会失败，执行`5.5步骤`即可解决。
cspherectl start
```

## 5.5 解决docker启动失败

在VMWare平台上，使用`IPVlan`网络模式`Docker`第一次启动会失败，需要执行以下命令解决。

```
net-plugin ip-range  --ip-start=172.17.0.1/24 --ip-end=172.17.0.254/24
```

查看cSphere相关服务进程是否已正常启动

```
# 查看和cSphere相关进程的运行状态
cspherectl status
```
如果在安装管理节点的时候ClusterSize设置的是3的话，需要将3台Agent都安装完成后，再生成COS页面上就可以看到已加入的Agent节点。

## 5.6 设置容器ip地址池

登录到任意一台`Agent`主机上，执行以下命令设置容器ip地址池

```
# 根据实际IP范围设定容器的IP地址池分配
net-plugin ip-range  --ip-start=192.168.1.30/24 --ip-end=192.168.1.254/24
```
## 5.7 结束安装

在生成COS验证码的页面，已看到加入的Agent节点，如不再添加计算节点，这个时候就可以点击结束安装了，页面也可以关闭了，此时您就可以使用cSphere平台了。

> **非常感谢您使用cSphere平台，使用过程中如有问题随时与我们联系！**

**联系邮箱**：*zcy@nicescale.com*

**联系电话**：*18511483965*