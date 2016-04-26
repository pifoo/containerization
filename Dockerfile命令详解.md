我们对Docker有了认识和了解，docker还有非常重要的一块，那就是docker镜像。提到docker镜像必不可少的一个文件是[Dockerfile](http://docs.docker.com/reference/builder/)
docker可以通过build命令自动从Dockerfile文件中读取并执行命令，最终生成docker镜像。使用Dockerfile非常简单，且重要！

# [Dockerfile](http://docs.docker.com/reference/builder/)命令详解

写文章时docker版本号为：v1.11.1-rc1 目前Dockerfile中共包含16个命令，文章内容详细介绍每个一个命令的**含义**和**用法**

## Dockerfile

格式：INSTRUCTION arguments

instruction 不区分大小写，推荐用大写，以免和其他参数冲突

## FROM

格式：
- `FROM \<image\>` 
- `FROM \<image\>:\<tag\>`
- `FROM \<image\>@\<digest\>`

Dockerfile中的第一条指令必须是FROM，FROM指令指定了构建镜像的base镜像。
`tag` or `digest`的值如果没有指定默认是latest. 

## MAINTAINER

格式：`MAINTAINER \<name\> \<Email\>`

编写维护Dockerfile的作者信息

## ENV

格式：
- `ENV \<key\>\<value\>`
- `ENV \<key\>=\<value\>`

在Dockerfile中如果有设置变量，可以直接调用；

> ${variable:-word} 如果没有给变量赋值，则变量的值默认为word
> ${variable:+word} 如果设置了变量的值，则word被覆盖，否则为空字符。
*注：环境变量转义使用\,如\$cSphere or \${cSphere}

```
FROM microimages/alpine
ENV docker /cSphere
WORKDIR ${docker}  # WORKDIR /cSphere
ADD . $docker      # ADD . /cSphere
COPY \$docker /carson # COPY $docker /carson
```

为容器声明环境变量，环境变量在子镜像中也可以使用。可以直接使用环境变量$variable_name
运行容器指定环境变量：`docker run --env <key>=<value>`

环境变量在以下命令中都可以使用：
`ADD COPY ENV EXPOSE LABEL USER WORKDIR VOLUME STOPSIGNAl`

## RUN

`RUN`有2种格式：
- `RUN \<command\>` (类似`/bin/sh -c`shell格式)

- `RUN ["executable", "param1", "param2"]` (exec格式)

**第一种** 使用shell格式时，命令通过`/bin/sh -c`执行; 

**第二种** 使用`exec`格式时，命令直接执行，容器不调用shell，并且`exec`格式中的参数会看作是`JSON`数组被Docker解析，所以要用(")双引号，不能用单引号(')。

举例：
`RUN [ "echo", "$HOME" ]`$HOME变量不会被替换,如果你想运行shell程序，使用：`RUN [ "sh", "-c", "echo", "$HOME" ]`

*注: 因为每执行一个Dockerfile的指令都会生成一个layer，所以在执行RUN的时候，一件事最好能在一个RUN中完成，例如：
```
RUN /bin/bash -c 'source $HOME/.bashrc ;\
echo $HOME'
```

## COPY

格式：COPY \<src\>\<dest\>

拷贝本地\<src\>目录下的文件到容器中\<dest\>目录。

## ADD

格式：
- ADD \<src\>\<dest\>
- ADD ["<src>",... "<dest>"]

`ADD hom* /mydir/`
从\<src\>目录下拷贝文件，目录或网络文件到容器的\<dest\>目录。和COPY非常相似，但比COPY功能多，拷贝的文件可以是一个网络文件，并且ADD有解压文件得功能。
*注：如果URL文件使用了authentication，请使用`RUN wget`,`RUN curl`,`ADD`不支持authentication。

## CMD

`CMD`有3种格式：

- `CMD \<commadn\> param1 param2`(shell格式)
- `CMD ["executable", "param1", "param2"]` （exex格式）
- `CMD ["param1", "param2"]`（为ENTRYPOINT命令提供参数）

虽然有三种格式，但在Dockerfile中如果有多个CMD命令，只有最后一个生效。

`CMD`命令主要是提供容器运行时得默认值。默认值，可以是一条指令，也可以是参数（ENTRYPOINT），`CMD`命令参数是一个动态值，信息会保存到镜像的JSON文件中。

举例：

```
ENTRYPOINT ["executable"]
CMD ["param1", "param2"]
```
启动容器执行：`["executable", "param1", "param2"]`
**注意：** 如果在命令行后面运行`docker run`时指定了命令参数，则会把`Dockerfile`中定义的`CMD`命令参数覆盖

## ENTRYPOINT

`ENTERPOINT`有2种格式：
- `ENTRYPOINT \<command\>` (shell格式)
- `ENTRYPOINT ["executable", "param1", "param2"]`（exec格式）

`ENTRYPOINT`和`CMD`命令类似，也是提供容器运行时得默认值，但它们有不同之处。同样一个Dockerfile中有多个ENTRYPOINT命令，只有最后一个生效。

- 当`ENTRYPOINT`命令使用\<commmand\>格式，`ENTRYPOINT`会忽略任何`CMD`指令和`docker run `传来的参数，直接运行在`/bin/sh -c`中;也就是说`ENTRYPOINT`执行的进程会是`/bin/sh -c`的**子进程**。所以进程的PID不会是1，而且也不能接受Unix信号。（执行docker stop $container_id的时候，进程接收不到SIGTERM信号）

- 使用`exec`格式，`docker run`传入的命令参数会覆盖`CMD`命令，并附加到`ENTRYPOINT`命令的参数中。推荐使用`exec`方式

`ENTRYPOINT`和`CMD`的区别：

```
||No ENTRYPOIT|ENTRYPOINT exec_entry p1_entry|ENTRYPOINT ["exec_entry" , "p1_entry"]|
|---|----|-----|------
|No CMD|error,not allowed|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry
|CMD ["exec_cmd" , "p1_cmd"]|exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry exec_cmd p1_entry|exec_entry p1_entry exec_cmd p1_cmd
|CMD ["p1_cmd" , "p2_cmd"]|p1_cmd p2_cmd|/bin/sh -c exec_entry p1_entry p1_cmd p2_cmd|exec_entry p1_entry p1_cmd p2_cmd
|CMD exec_cmd p1_cmd|/bin/sh -c exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd|exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd
 ```

## ONBUILD

格式：`ONBUILD [INSTRUCTION]`

`ONBUILD`指令的功能是添加一个将来执行触发器指令到镜像中。当Dockerfile中`FROM`的镜像中包含`ONBUILD`指令，在构建此镜像的时候会触发`ONBUILD`指令。但如果当前Dockerfile中存在`ONBUID`指令，不会执行就。`ONBUILD`指令在生成**应用镜像**时用处非常大。

**`ONBUILD`如何工作**

1.构建过程中，`ONBUILD`指令会添加到触发器指令镜像元数据中。触发器指令不会在当前构建过程中生效
2.构建完后，触发器指令会被保存到镜像的详情中，主键是`OnBuild`，可以使用`docker inspect`命令查看到
3.之后此镜像可能是构建其他镜像的父镜像，在构建过程中，`FROM`指令会查找`ONBUILD`触发器指令，并按照之前定义的顺序执行；如果触发器指令执行失败，则构建新镜像失败并退出；如果触发器指令执行成功，则继续往下执行。
4.构建成功后`ONBUILD`指令清除，固不会被孙子辈镜像继承。

## VOLUME

格式：`VOLUME ["/data"]`

为docker主机和容器做目录映射，volume目录信息会保存到镜像的JSON文件中，在运行`docker run`命令时指定`$HOST_DIR`

## LABEL

格式：`LABEL <key>=<value> <key>=<value> <key>=<value> ...`

LABEL指令，添加一个元数据到镜像中。一个镜像中可以有多个标签，建议写一个（因为每多个LABEL指令镜像会多一个`layer`）`LABEL`是一个键值对\<key\>\<value\>，在一个标签值,包括空间使用引号和反斜杠作为您在命令行解析。
举例：
`LABEL multi.label1="value1" multi.label2="value2" other="value3"`

## EXPOSE
格式：`EXPOSE <port> [<port>...]`

在Dockerfile中定义端口，默认是不往外暴露，在运行`docker run ` `-p` or `-P`暴露

## USER

格式：`USER daemon`

设定一个用户或者用户ID,在执行`RUN``CMD``ENTRYPOINT`等指令时指定以那个用户得身份去执行

## ARG

格式：`ARG <name>[=<default value>]`

`ARG`定义一个变量，用户可以在执行`docker build`命令时使用`--build-arg <varname>=<value>`,如果用户指定的参数不是在`Dockerfile`中定义的，则输出以下错误信息：`One or more build-args were not consumed,failing build.`

例子：
```
FROM microimages/alpine
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER
```
构建镜像`docker build --build-arg CONT_IMG_VER=v2.0.1 Dockerfile`

在Dockerfile中`RUN`取的变量的值为`v1.0.0`,但是`ARG`取得是设置的值`v2.0.1`,如果执行`docker build Dockerfile`那么`CONT_IMG_VER`的值就还是`v1.0.0`

## WORKDIR

格式：`WORKDIR /path/to/workdir`

当执行`RUN``CMD``ENTRYPOINT``ADD``CMD`等命令，设置工作目录

## STOPSIGNAL

格式：`STOPSIGNAL signal`

`STOPSIGNAL`指令：发送signal信号给容器，让容器退出。

## .dockerignore

如果在Dockerfile文件目录下有.dockerignore文件，docker在构建镜像的时候会把在.dockerignore文件中定义的文件排除出去

```
    */temp*
    */*/temp*
    temp?
    *.md
    !LICENSE.md
```


