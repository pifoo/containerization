
## 控制中心HA部署:

### 1. 安装3台centos7

#### 1.1 安装centos7 64bit
Centos 7 64bit Minimal镜像安装, 下载地址:
```liquid
http://centos.ustc.edu.cn/centos/7.2.1511/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso
```
系统安装尽量选择默认`英文`语言进行安装

#### 1.2 基本设置
```bash
// 主机名设置, 根据实际角色设定, 比如`c1` `c2` `c3`
# echo "c1" > /etc/hostname

// 关闭selinux
# setenforce 0
# sed -i '/^SELINUX=/cSELINUX=disabled' /etc/selinux/config
```

#### 1.3 安装依赖包和4.6内核
```bash
安装csphere yum repo
# curl -Ss http://52.68.20.57/pubrepo/centos/7/x86_64/csphere.repo > /etc/yum.repos.d/csphere.repo
# yum repolist csphere

安装依赖的软件包:
# yum -y --disablerepo='*' --enablerepo=csphere install bridge-utils net-tools psmisc subversion git fuse ntp

安装4.6.0内核:
# yum -y --disablerepo='*' --enablerepo=csphere install kernel-ml-4.6.0

使用新内核启动:
# grub2-set-default 0
# reboot
# uname -r
4.6.0-1.el7.elrepo.x86_64
```

注：  
如果服务器所在网络连不上公网，可以提前把所有rpm包提前下载到本地电脑再向服务器上传：
```bash
# wget -r -np -nH -R "index.html*" http://52.68.20.57/pubrepo/centos/7/x86_64/
```

#### 1.4 Docker数据分区
需要一个单独的数据分区存放docker数据, 并  
使用参数`-n ftype=1`格式化为xfs文件系统, 否则在4.6内核上无法正常创建容器，  
使用参数`prjquota`挂载， 否则无法正常使用容器的磁盘空间配额功能。
```bash
# fdisk /dev/vdb                  // 新设备上建立分区
# mkfs.xfs -n ftype=1 /dev/vdb1   // 格式化
# mkdir /docker-data/
# echo "/dev/vdb1 /docker-data xfs defaults,prjquota 0 0" >> /etc/fstab
# mount -a
# mkdir -p /docker-data/docker
# ln -sv /docker-data/docker /var/lib/docker
```

### 2. 安装Controller:
在`所有三台`机器上执行:
```bash
# rpm -ivh csphere-controller-1.4.4-rhel7.x86_64.rpm 
```

### 3. 初始化Controller参数
在`所有三台`机器上执行:
```bash
# Role=controller Port=80 ClusterSize=1 MongoRepl=YES  csphere_init
+OK

参数说明:
`Port`:    管理页面HTTP服务端口
`ClusterSize`:  Etcd集群大小, 此处设置的值就是Agent最小的安装数目. 如ClusterSize=3,则最少需要安装3台Agent来完成Etcd集群初始化.
```

### 4. 单独启动mongodb
在`所有三台`机器上执行:
```bash
# cspherectl  start mongodb
```

### 5. 配置mongodb replset集群:
在`其中一台`机器上执行:
```bash
连接本地mongodb:
# mongo

设置集群参数:  (根据实际情况修改IP地址)
> rs.initiate({ _id: "rs0", version: 1, members: [ { _id: 0, host: "192.168.122.5:27017"}, { _id: 1, host: "192.168.122.6:27017"}, { _id: 2, host: "192.168.122.7:27017"}  ] });
{ "ok" : 1 }

查看集群状态: (默认第一个IP是主节点`PRIMARY`, 另外两个IP是`SECONDARY`)
> rs0:PRIMARY> rs.status();

退出
> rs0:PRIMARY> exit
bye
```

mongodb repl集群配置完毕.

### 6. 启动其他组件
在`所有三台`机器上执行:
```bash
# cspherectl  start etcd
# cspherectl  start docker
# cspherectl  start monitor  
# systemctl   enable csphere-monitor  // 开机激活 monitor 服务

说明: 不要手工执行启动 `prometheus` `controller` `agent` 这三个组件了,
这三个组件的启动停止由 `monitor` 根据 `monogodb` 的角色状态负责启动停止.
 
```

以上命令执行完毕后, 只有 `mongodb` 主节点所在的机器上运行了所有服务 (`cspherectl s` 查看) 
至此, 主控HA部署完毕.  可以web登录主节点的IP, 完成帐号初始化, 填写license



## 节点部署:

### 1. 安装CentOS7系统:
说明: 如果安装主控的时候, 设置的`ClusterSize`值是多少, 则Agent首次部署就至少需要安装`ClusterSize`台.

#### 1.1 系统安装
操作系统的安装部署请参考主控中心安装的 `1.1` `1.2` `1.3` `1.4` 步骤进行安装和设置.

#### 1.2 建立br0网桥 
如果容器使用bridge网络则此步骤必须, ipvlan网络可跳过.
```bash
# cd /etc/sysconfig/network-scripts
# cat ifcfg-br0 

DEVICE=br0
TYPE=Bridge
ONBOOT=yes
IPADDR=192.168.122.12  // IP地址,根据实际填写
NETMASK=255.255.255.0  // 掩码
GATEWAY=192.168.122.1  // 默认网关
DNS1=192.168.122.1     // 主DNS
BOOTPROTO=static
NM_CONTROLLED=no

# cat ifcfg-eno16777736  // 假设物理网卡名为 eno16777736
TYPE=Ethernet
DEVICE=eno16777736
NAME=eno16777736
ONBOOT=yes
BRIDGE=br0
NM_CONTROLLED=no

# reboot                 // 重启系统
# ifconfig br0           // 确认IP地址在br0上
# brctl show br0         // 确认br0
bridge name	bridge id		STP enabled	interfaces
br0		8000.525400e5ef57	no		eth0
```

### 2. 安装Agent:
```bash
centos7系统自带的iproute版本旧,需要替换到4以上的版本,否则容器无法正常启动
# rpm -Uvh http://demo.csphere.cn/pubrepo/centos/7/x86_64/iproute-4.1.1-3.fc23.x86_64.rpm --nodeps
# rpmdb --rebuilddb

# rpm -ivh csphere-agent-1.4.4-rhel7.x86_64.rpm
```

### 3. 初始化Agent参数:
可以使用`ipvlan`或者`bridge`网络模式初始化， 请选择一种进行操作：
```bash
使用ipvlan模式初始化：
# Role=agent ControllerAddr=192.168.122.10:80,192.168.122.11:80,192.168.122.12:80 InstCode=6906 NetMode=ipvlan InetDev=eth0 csphere_init 
+OK

或者

使用bridge模式初始化：
# Role=agent ControllerAddr=192.168.122.10:80,192.168.122.11:80,192.168.122.12:80 InstCode=6906 NetMode=bridge csphere_init

参数说明:
`ControllerAddr`:  主控中心的地址:端口, 多个主控中心用`,`分隔, 将当前运行的主控中心`主节点`写在第一个.
`InstCode`:  安装码, 到主控中心页面上生成
`NetMode`:   Docker容器网络模式, ipvlan或bridge
`InetDev`:   物理网卡名, 只有ipvlan才需要这个参数
```

### 4. 启动Agent:
```bash
# cspherectl  start
```
说明: 如果NetMode=ipvlan的话, 可能`docker`会启动失败, 没关系.
执行如下命令后, `docker` 即可恢复正常运行
```bash
# net-plugin ip-range  --ip-start=172.17.0.1/24 --ip-end=172.17.0.254/24
```

### 5. 设置容器使用的IP地址范围:
如不设置无法正常启动容器.
```bash
根据实际IP范围设定容器的IP地址池分配
# net-plugin ip-range  --ip-start=192.168.122.30/24 --ip-end=192.168.122.50/24
```

## 主控HA切换测试:
 - 将主控集群中的`主节点`关机.
 - 观察主控集群中的另外`两个副节点`,有一个会接管成为`主节点`继续运行`controller`,并对外提供80服务
 - 观察Agent运行状态正常. 日志中有 `health controller is xxxx` `switch controller to xxxx`
