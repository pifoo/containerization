# PHP项目Docker化指南

> Hi， 大家好。今天和大家分享一下如何使用将一个PHP项目Docker化并进行自动镜像构建和自动部署的思路，希望对大家有帮助。

## 快速上手

PHP官方在 hub.docker.com 上维护了官方的PHP Docker镜像，包含了从PHP 5.5到7.0的多种不同版本的镜像。

![php-images-version](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/php-image-version.png)

我们将以PHP官方的Docker镜像为基础，介绍如何将一个简单的PHP应用Docker化。

- 创建一个新目录 php-quickstart，作为我们的项目目录

- 在项目目录下创建文件 app.php

```
<?php
  echo “Hello Docker!”
?>
```

- 在项目目录下创建文件 Dockerfile

```
FROM php:5.6-cli

COPY . /project
WORKDIR /project
CMD ["php", "./app.php"]
```

上述 Dockerfile 中，通过 `FROM` 指令，我们将官方的 `php-5.6-cli` 作为我们的基础镜像。

通过 `COPY` 指令，我们把当前目录下的文件，复制到镜像的 `/project` 目录

`CMD` 指令设置了镜像默认执行的命令，`WORKDIR` 则是设置了镜像执行命令时的目录

- 构建镜像

`docker build -t php-app .`

这将会生成一个名为 `php-app` 的镜像

- 运行容器

`docker run php-app`

这时候，容器将会执行我们之前创建的 `app.php`， 并输出：

`Hello Docker!`

## PHP + MySQL 的Docker化示例

接下来，我们通过一个 PHP + MySQL 的例子，介绍 PHP 应用 Docker 化之后，如何连接数据库。

- 创建一个新的目录 php-mysql 作为我们的项目目录

- 创建一个新的目录 php-mysql 作为我们的项目目录

```
<?php
 $mysql = new mysqli('db', 'root', $_ENV['MYSQL_ROOT_PASSWORD']);
 echo 'Connected to mysql: '.$mysql->host_info;
?>
```

在 `index.php` 中，我们的 PHP 应用将会通过主机名称 `db` 连接到 mysql 数据库，同时使用用户名 `root`， 以及环境变量中的 `MYSQL_ROOT_PASSWORD`对数据库进行连接。这里简单地通过echo 连接信息来确认 MySQL 连接是否正常。

- 在项目目录下创建 Dockerfile

```
FROM php:5.6-apache
RUN docker-php-ext-install mysqli
COPY . /var/www/html
```

这里我们使用的是官方的 `php:5.6-apache` 镜像，因为我们这一次希望可以直接从浏览器访问这个 PHP 应用。

另外我们通过 `RUN` 指令运行 `docker-php-ext-install mysqli` 额外安装了PHP的`mysqli`扩展

- 构建镜像

`docker build -t php-mysql-app .`

- 创建 MySQL 容器

`docker run --name db -e MYSQL_ROOT_PASSWORD=secret -d mysql:5.6`

我们在这里使用官方的 `mysql:5.6` 镜像创建了一个 MySQL 的容器

`--name` 参数将容器命名为 `db`

`-e MYSQL_ROOT_PASSWORD=secret` 通过环境变量，我们将 MySQL 的 root 用户密码设置为 `secret`

`-d` 参数将这个容器设置为后台运行

- 启动 PHP 容器，并将其连接到 MySQL 容器

`docker run --link db -e MYSQL_ROOT_PASSWORD=secret -p 8080:80 php-mysql-app`

我们运行了之前构建的 `php-mysql-app` 镜像，并将上一步创建的 `mysql-instance` 这个MySQL容器和它连接，同时我们把MySQL的root密码通过环境变量`MYSQL_ROOT_PASSWORD` 传到容器内部`-p 8080:80` 将容器的 `80` 端口映射到了主机的 `8080` 端口

- 从浏览器访问 http://127.0.0.1:8080

```
Connected to mysql: db via TCP/IP
```

我们将得到从浏览器得到 `index.php` 的执行结果。

## 基于cSphere 私有Docker Registry的镜像自动构建

在一个Docker化的项目中，项目的Docker镜像成为了项目交付的最终元件。因此在项目的持续集成和持续交付环节中，镜像的自动构建是必不可少的一个环节。

这里介绍如何利用cSphere的私有镜像仓库配置镜像自动构建，实现在代码Push到仓库之后，自动构建Docker镜像。

- 创建私有Docker Registry

  在通过cSphere的镜像仓库页面，点击`新建镜像仓库`按钮，根据提示即可成功创建一个私有的镜像仓库.

![create-registry](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/create-registry.png)

- 配置项目

  进入上一步创建的镜像仓库页面，点击`添加自动构建`按钮，填写项目的 Git 仓库地址和Dockerfile路径：

![add-auto-build](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/add-auto-build.png)

然后根据提示，设置镜像构建后，在镜像仓库的存放位置，和需要进行自动构建的分支：

![setting-brach](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/seeting-brach.png)

- 设置项目的Web Hook和 Deploy key

![setting-deploy-key](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/seeting-deploy-key.png)

根据提示，为项目设置好Webhook和Deploy Key. 这样当项目有新的代码push到上一步中设置的分支之后，私有Docker Registry就会进行镜像的自动构建, 在构建成功之后，自动将镜像Push到镜像仓库的指定位置

![auto-build](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/auto-build.png)

## 使用cSphere部署和管理PHP应用

在实现了自动构建项目的镜像之后，接下来我们来看如何通过cSphere快速将会项目部署到各种环境中。

- 创建应用模板

  进入cSphere的应用模板页面，点击`创建新模板`按钮，根据提示新建一个应用模板

![create-template](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/create-template.png)

- 添加MySQL服务

  在之前的PHP + MySQL 项目Docker化示例中，我们通过以下的命令启动了MySQL容器:

```
docker run --name db -e MYSQL_ROOT_PASSWORD=secret -d mysql:5.6
```

这时我们把上述命令配置成应用模板中的一个服务：`db`

![add-db-service](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/add-db-service.png)

同时设置好环境变量

![setting-environment](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/setting-environment.png)

- 添加PHP服务

  在 PHP + MySQL 项目Docker化示例中，我们通过以下的命令启动了PHP容器:

```
docker run --link db -e MYSQL_ROOT_PASSWORD=secret -p 8080:80 php-mysql-app
```

  同时，我们在自动构建镜像中，设置了自动构建镜像为 `192.168.1.130/tsing/php-mysql-app:latest`
  这里我们把上述信息配置成应用模板的另一服务：`php`

![create-php-service](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/create-php-service.png)

  设置PHP代码中使用的环境变量值

![setting-php-environment](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/setting-php-environment.png)

  `--link db` 这个参数无需在应用模板中设置，因为cSphere应用管理会自动根据服务的名称，自动处理不同容器的连接关系。
  `-p` 端口映射也不需要设置，因为cSphere应用管理创建的容器都有独立的IP，不再需要把容器的端口映射到主机上

- 保存模板

![save-template](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/save-template.png)

- 部署应用

  点击上一步刚刚创建成功的模板版本，最右边的部署按钮，便可以开始进行部署。

![deploy-application](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/deploy-application.png)

在这个界面中，你可以选择将应用部署到哪一个主机分组中, 可以根据需要，把应用部署到开发、测试、生产不同环境的主机上。当然，也可以在一个环境部署多个实例, 这些实例之间是互相隔离的。

![denploy-instance](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/deployed-instance.png)

- 应用模板管理

  在应用模板页面，你可以对应用模板进行修改，每次模板的修改都会产生一个新的版本，方便进行升级和回滚。

![application-template](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/application-template.png)

- 应用管理
 
  在应用实例的页面，你可以对应用实例进行管理, 对应用的服务进行扩容，重启
![application-instance](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/instance-list.png)

  点击升级 `· 回滚按钮`，可以快速将应用更新至指定版本的模板

![update-and-rollback](https://github.com/billycyzhang/Shell/blob/master/php-containerization/images/update-and-rollback.png)

## 应用部署自动化

当镜像重新构建之后，可以在 cSphere 面板上点击服务的`重新部署`按钮来升级服务, 也可以直接使用 cSphere 的 API 来实现自动化升级。

在调用cSphere的API前，请先在cSphere的设置页面生成一个API Key:

调用以下 API，即可实现自动升级应用

```
curl -H 'cSphere-Api-Key: 6bbdc50dd0561b47ca8186f8ac29acde70bc65b3' \
-X PUT http://192.168.1.130/api/instances/php-mysql-example/redeploy	
```

今天介绍的内容就这些。希望能为大家使用Docker来改进项目开发、测试和交付流程有所帮助和借鉴意义，谢谢。
