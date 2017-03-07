# 4.安装cSphere Controller

## 4.1 安装cSphere-Controller节点

*前提*：将`cSphere-controller`的`rpm`包上传至服务器

```
rpm -ivh csphere-controller-1.5.6-rhel7.x86_64.rpm 
```
## 4.2 初始化Controller

执行以下命令之前请先阅读**特别提示**：

```
Role=controller ClusterSize=1 Port=80 MongoRepl=NO csphere_init
```
**特别提示**：以上命令需要注意的参数有,`ClusterSize`,`Port`,`MongoRepl`

```
参数解析：
ClusterSize:是Agent节点中的Etcd服务集群节点数，测试环境设置成1或3，如果设置成3 Agent节点要将3台**都安装完毕后集群才能建成**。

*demo环境可以设置成1*

Port：通过浏览器访问管理节点的Web页面的Http端口,标准配置是80。
MongoRepl:是否将控制器部署成集群模式
```
# 4.3 启动Controller

```
# cspherectl为内置命令可直接执行
cspherectl  start
```
# 4.4 添加企业版license

## 4.4.1 服务状态
待执行完`cspherectl start`命令后，管理节点上的服务均正常启动，运行状态为`running`。

## 4.4.2 访问Controller节点Web页面

打开浏览器访问:`192.168.1.2`,即可访问到Controller的Web页面。

## 4.4.3 添加用户

安装完成后，需要注册第一个用户，并设置密码至少`8`位

## 4.4.4 申请license

请与`希云cSphere`工作人员联系，获取**企业版`license`**;

**联系邮箱**：*zcy@nicescale.com*

**联系电话**：*18511483965*

## 4.4.5 将企业版license添加至平台

拿到企业版license后，点击左侧菜单栏`系统设置`-->`License`-->`更换License`-->`添加`

> 到此`管理节点`安装完毕！