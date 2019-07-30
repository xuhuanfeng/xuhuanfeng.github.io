---
title: Docker学习笔记
date: 2019-07-30 15:44:00
tags: Docker
categories: 容器
---

# Docker介绍及基本使用

## Docker介绍

Docker是新的容器化技术，相对于虚拟机技术，更加轻巧，更加易于移植。

<!--more-->

Docker特征

- 速度快
- 隔离机制
- 低消耗


Docker核心组件

- Docker客户端和服务器
  - Docker是一个客户端-服务器(C/S)架构的应用程序
  - Docker提供了一个命令行工具以及完整的RESTful API
  - 可以在同一个宿主机上运行Docker守护进程和客户端，也可以从本地Docker客户端连接到另一台宿主机上的Docker守护进程
- Docker镜像
  - 镜像是构建Docker世界的基石
  - 用于基于镜像来运行自己的容器
  - 镜像是基于联合文件系统的一种层式结构，由一系列指令一步步构建出来
  - 镜像体积很小，易于分享、存储和更新
- Registry(仓库)
  - Docker用Registry来保存用户构建的镜像
  - Registry分为公共(如Docker Hub)和私有(自建仓库等)
- Docker容器
  - 容器是基于镜像启动的
  - 一个容器中可以运行一个或多个进程
  - 每个容器包含一个软件镜像
  - 可以简单地理解为容器就是运行起来的镜像

Docker应用场景

- 加速本地开发和构建流程(容器可以直接在不同的环境中运行)
- 能够让独立服务或应用程序在不同的环境中，得到相同的运行结果
- 使用Docker创建隔离的环境来进行测试
- 构建一个多用户的PaaS基础设施
- 为开发、测试提供一个轻量级的独立沙盒环境
- 提供SaaS应用程序
- 高性能、超大规模的宿主机部署

## Docker安装

这里演示使用的os是centos7 64位，需要使用root运行

Docker必备环境

- 运行64位CPU架构的计算机
- 运行Linux3.8及以上版本内核

上述内容可以通过`uname -r`命令进行查看

```shell
3.10.0-862.14.4.el7.x86_64
```

可以看到这里的内核版本是3.10，机器是64为，满足安装Docker的必备环境。

安装Docker，命令为`yum install docker`

依赖分析后的内容如下

```shell
Dependencies Resolved

===============================================================================================
 Package                             Arch   Version                              Repository
                                                                                          Size
===============================================================================================
Installing:
 docker                              x86_64 2:1.13.1-75.git8633870.el7.centos    extras   16 M
Installing for dependencies:
 PyYAML                              x86_64 3.10-11.el7                          base    153 k
 atomic-registries                   x86_64 1:1.22.1-25.git5a342e3.el7.centos    extras   35 k
 audit-libs-python                   x86_64 2.8.1-3.el7_5.1                      updates  75 k
 checkpolicy                         x86_64 2.5-6.el7                            base    294 k
 container-selinux                   noarch 2:2.68-1.el7                         extras   36 k
 container-storage-setup             noarch 0.11.0-2.git5eaf76c.el7              extras   35 k
 device-mapper-event                 x86_64 7:1.02.146-4.el7                     base    185 k
 device-mapper-event-libs            x86_64 7:1.02.146-4.el7                     base    184 k
 device-mapper-persistent-data       x86_64 0.7.3-3.el7                          base    405 k
 docker-client                       x86_64 2:1.13.1-75.git8633870.el7.centos    extras  3.8 M
 docker-common                       x86_64 2:1.13.1-75.git8633870.el7.centos    extras   93 k
 libcgroup                           x86_64 0.41-15.el7                          base     65 k
 libsemanage-python                  x86_64 2.5-11.el7                           base    112 k
 libyaml                             x86_64 0.1.4-11.el7_0                       base     55 k
 lvm2                                x86_64 7:2.02.177-4.el7                     base    1.3 M
 lvm2-libs                           x86_64 7:2.02.177-4.el7                     base    1.0 M
 oci-register-machine                x86_64 1:0-6.git2b44233.el7                 extras  1.1 M
 oci-systemd-hook                    x86_64 1:0.1.17-2.git83283a0.el7            extras   33 k
 oci-umount                          x86_64 2:2.3.3-3.gite3c9055.el7             extras   32 k
 policycoreutils-python              x86_64 2.5-22.el7                           base    454 k
 python-IPy                          noarch 0.75-6.el7                           base     32 k
 python2-pytoml                      noarch 0.1.18-1.el7                         epel     20 k
 setools-libs                        x86_64 3.3.8-2.el7                          base    619 k
 skopeo-containers                   x86_64 1:0.1.31-1.dev.gitae64ff7.el7.centos extras   17 k
 subscription-manager-rhsm-certificates
                                     x86_64 1.20.11-1.el7.centos                 base    195 k
 yajl                                x86_64 2.0.4-4.el7                          base     39 k

Transaction Summary
===============================================================================================Install  1 Package (+26 Dependent packages)

Total download size: 27 M
Installed size: 87 M
Is this ok [y/d/N]:
```

直接输入`y`即可

检查是否安装成功：`docker --version`，输入内容为 `Docker version 1.13.1, build 8633870/1.13.1`

启动docker服务：`systemctl start docker`

设置开机自启：`systemctl enable docker`

检查docker是否启动成功：`docker info`，输入内容如下，则表示启动成功

```shell
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
...
```

关闭Docker：`systemctl stop docker`

重启Docker：`systemctl restart docker`

## Docker使用

### 容器管理

- 部署容器(容器只有先部署才能启动跟使用，部署即从镜像到容器的过程)
  - 命令：`docker run --name INSTANCE_NAME -d DOCKER_IMAGE_NAME[:TAG]`
  - 演示案例
    - 命令：`docker run --name dkr_tomcat [-d] [-p LOCAL_PORT:CONTAINER:PORT] tomcat`，没有指定tag则为最新
      - `-d` 表示以守护进程的形式启动(通常都是需要的)
      - `-p` 指定本地以及容器的端口，也可以直接指定容器的端口
    - 如果本地没有指定的镜像，则会先从docker hub中下载，如下所示
        ```shell
        Unable to find image 'tomcat:latest' locally
        Trying to pull repository docker.io/library/tomcat ...
        latest: Pulling from docker.io/library/tomcat
        ...
        Status: Downloaded newer image for docker.io/tomcat:latest
        36322b0ba8389b058aa3bcb82442244931945ac2d91186615a0a4ed124053655
        ```
    - 部署成功之后，会输出一段ID，如这里的`36322b0ba8389b058aa3bcb82442244931945ac2d91186615a0a4ed124053655`
- 查看运行的容器
  - 命令：`docker ps`
      ```shell
        CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
        36322b0ba838        tomcat              "catalina.sh run"   3 minutes ago       Up 3 minutes        8080/tcp            dkr_tomcat
      ```
    - 使用`docker ps -a`可以查看所有的容器
- 停止容器

  - 命令：`docker stop ID/NAME`
- 启动容器

  - 命令：`docker start ID/NAME`，该命令是运行一个已经停止的容器
- 重启容器

  - 命令：`docker restart ID/NAME`
- 查看容器日志
  - 命令：`docker logs ID/NAME`
  - 可以使用`-f`选项来启动日志监控(直接在屏幕打印日志，没有则阻塞)，选项意义同`tail`命令，`ctrl-c`退出
- 查看容器内进程

  - 命令：`docker top ID/NAME`
- 在容器内部运行进程
  - 命令：`docker exec [OPTION] ID/NAME COMMAND`
     `-d`表示在后台执行该命令
     - `-i` 表示进入交互模式
     - `-t` 表示创建模拟tty
   - 案例：
     - 命令`docker exec -it dkr_tomcat /bin/bash`
     - 表示指定/bin/bash命令并且进入交互模式，此时可以在容器中执行一些命令，如`pwd`，`ls`等
- 查看更多容器信息

  - 命令：`docker inspect ID/NAME`
- 删除容器
  - 命令：`docker rm ID/NAME`
  - 运行中的容器需要先停止才能删除

### 镜像管理

Docker镜像是由文件系统叠加而成，最底层是一个引导文件系统bootfs，当容器启动后，会被移到内存中，而引导文件系统则会被卸载

Docker镜像第二层是rootfs，位于引导文件系统之上，可以是一种或多种操作系统。

在Docker中，一个镜像可以放到另一个镜像的顶部，位于下面的镜像称为父镜像，最底部的镜像称为基础镜像，当从一个镜像中启动容器之后，Docker会在该镜像的最顶层加载一个读写文件系统

- 查看镜像
  - 命令：`docker images`
  - 本地镜像都保存在`/var/lib/docker`目录下(在具体的存储驱动目录下)
  - 容器则存储在`/var/lib/docker/containers`目录下
- 拉取镜像

  - 命令：`docker pull IMAGE_NAME[:TAG]`，不指定TAG则会拉取仓库内全部镜像
- 查找镜像

  - 命令：`docker search IMAGE_NAME`
- 删除镜像

  - 命令：`docker rm IAMGE_NAME`
- 构建镜像，一般通过Dockerfile以及`docker build`命令来构建镜像
  - Dockerfile由一系列指令和参数组成，每条指令必须大写，后面必须跟一个参数，如`FROM `
  - 每条指令都会创建一个新的镜像层并对镜像进行提交
  - `#`开头为注释内容
  - Dockerfile的第一条指令必须为`FROM`，后续指令都将基于该镜像进行，该镜像也称为基础镜像
  - `RUN`指令用于在当前镜像中运行指定的命令，默认使用shell来执行，可以通过`['cmd', 'arg', '...']`来避免
  - `EXPOSE`指令用于指定容器使用的端口
  - 案例
    - 新建文件夹`static_web`
    - 编辑Dockerfile 

        > Dockerfile

        ```shell
        # Version 0.0.1
        # 命令大小
        FROM nginx:latest
        RUN echo "Hi I am Xavier" >> /usr/share/nginx/html/index.html
        EXPOSE 80
        ```

    - 然后执行命令：`docker build -t="xavier/static_web" .`
      - `-t`用于指定仓库和名称
      - 注意后面的`.`，指定当前目录(Dockerfile所在目录)，如果是在外面，需要注意目录的路径。
    - 输出内容

        ```shell
        Sending build context to Docker daemon 2.048 kB
        Step 1/3 : FROM nginx:latest
        ---> 62f816a209e6
        Step 2/3 : RUN echo "Hi I am Xavier" >> /usr/share/nginx/html/index.html
        ---> Running in 755840b827eb
        
        ---> c177cbb50afe
        Removing intermediate container 755840b827eb
        Step 3/3 : EXPOSE 80
        ---> Running in 200168bc6cdf
        ---> 89dbc87994b6
        Removing intermediate container 200168bc6cdf
        Successfully built 89dbc87994b6
        ```


### Dockerfile指令

- ADD，将构建目录下指定文件添加到镜像中， `ADD a.file /usr/local/share/a.file，`如果是压缩文件，会自动解压
- COPY， 仅仅复制，不会自动提取或解压

## Docker容器互连

默认情况下容器是无法被外界访问的，可以使用暴露端口的方式，但是该方式不推荐，更合适的方式是通过绑定机制来实现容器之间的互连，比如一个tomcat容器以及一个redis容器，可以将tomcat容器与redis容器进行绑定，绑定之后只有tomcat容器能访问到redis容器，其他的容器无法访问，当然，外界环境同样无法访问啦。

绑定的方式，只需要在部署容器的时候，指定一下所要绑定的容器即可，`docker run --link TARGET_CONTAINER:ALIAS_NAME IMAGE`，如`docker run --link redis:redis-db tomcat`，然后可以在tomcat容器中的`/etc/hosts`文件看到，直接绑定了`172.17.0.3  redis-db d78523d89091 redis`，可以通过在tomcat容器中执行`ping redis-db`检测两个容器之间的通信情况。

## 卷挂载

由于Docker容器重启之后，容器中的数据会丢失，为了保存数据，Docker中提供了卷的概念，可以通过卷在外部文件和容器文件进行映射，从而达到共享数据或者持久化数据的目的。

在构建容器时，可以通过`-v SOURCE_DIR:TARGET_DIR`进行挂载。

