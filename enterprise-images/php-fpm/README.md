# Nginx + PHP-FPM

> nignx_version:1.6.3; php_version:5.4.16

## 基于[zcy/centos7](https://github.com/billycyzhang/odt/tree/master/centos7)镜像构建

`FROM zcy/centos7`

## 网站根目录`/app`

网站根目录下默认添加了`index.html`、`info.php`、`logo.png`文件。

## 使用supervisor管理`nginx`、`php-fpm`服务

```
COPY supervisor_nginx.conf /etc/supervisor/confd/nginx.conf
COPY supervisor_php-fpm.conf /etc/supervisor/confd/php-fpm.conf
```

## nginx启动在前端

nginx启动在前端，default.conf添加php-fpm，以及网络根目录修改为/app.

>联系邮箱: “zcy@nicescale.com”  weixi: "kernel0963"