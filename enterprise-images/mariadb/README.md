# Mariadb 数据库

> mariadb_version:5.5.47

## 优化Mariadb数据库

`COPY my.cnf /etc/my.cnf`

## 设置登陆数据库密码

`docker run -d -p 3306:3306 --name mydb -e DB_USER=admin -e DB_USER=mypaas zcy/mariadb:5.5.47`

## 设置远程登陆数据库密码

`docker run -d -p 3306:3306 --name mydb -e DB_USER=admin -e DB_USER=mypaas -e REMOTE_ADMIN=true -e REMOTE_ADMIN_USER=zcy -e REMOTE_ADMIN_PASS=zcypaas zcy/mariadb:5.5.47`

## 数据保护

`docker run -d -p 3306:3306 --name mydb -e DB_USER=admin -e DB_USER=mypaas -e REMOTE_ADMIN=true -e REMOTE_ADMIN_USER=zcy -e REMOTE_ADMIN_PASS=zcypaas -v /mydata:/var/lib/mysql zcy/mariadb:5.5.47`

> 容器删除后数据不被删除，新创建容器数据目录如挂载到`/mydata`目录下，原来的数据可继续使用。

> 联系邮箱: “zcy@nicescale.com”  weixi: "kernel0963"