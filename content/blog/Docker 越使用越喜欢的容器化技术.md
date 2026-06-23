+++
title = "Docker：越使用越喜欢的容器化技术"
date = "2025-06-19T20:35:09+08:00"
tags = ["Docker"]
status = "landing"
+++

> Reference：[40分钟的Docker实战攻略](https://www.bilibili.com/video/BV1THKyzBER6)

Docker 是目前最成熟高效的应用服务容器化部署技术，简单来说就是用容器化技术给应用程序封装独立的运行环境，每个运行环境就是一个容器，运行容器的计算机就是宿主机。

## Docker vs 虚拟机

Docker 容器与虚拟机的最大区别就是 Docker 容器之间共用同一个系统内核，而每个虚拟机都包含一个操作系统的完整内核。

所以 Docker 容器比虚拟机更小、更轻量化，启动速度更快

## 镜像与容器

Docker 最核心的两个概念就是镜像和容器，镜像是容器的模板，就像软件的安装包，而容器是安装出来的软件。我们可以将安装包分享给别人让别人也可以安装我们的软件。

另一个概念是镜像仓库，就是用来存储和分享 Docker 镜像的仓库，每个人都可以上传分享自己的镜像，也可以从仓库拉取别人上传分享的镜像。Docker 官方的镜像仓库是[Docker Hub](https://hub.docker.com/):

<img src="https://cdn.kmoon.fun/2026/2026-06-20T08-13-16-039Z.png" alt="" width=500/>

Docker 通常来说是基于 Linux 的容器化技术，Windows 和 Mac 系统都是虚拟了一个 Linux 子系统来运行 Docker。

## Docker 命令

### docker pull

跟我们使用的 git pull 命令差不多，用于从镜像仓库拉取镜像。

```docker
# docker pull 仓库地址/命名空间/镜像名称:标签（版本号）

docker pull docker.io/library/nginx:latest
```

### docker images

列出所有下载的镜像

### docker rmi

remove image，即删除镜像，后面跟镜像名称或者镜像 ID。

### docker run

使用镜像创建并运行容器，后面跟镜像名称或 ID。

- -d 后台运行，不占用当前终端。
- -p 端口映射（宿主机端口：容器内端口）。
- -v 目录挂载（宿主机目录：容器内目录），用于数据持久化。第一次运行宿主机目录会覆盖容器目录。
- -e 传递环境变量
- --name 给容器命名
- -it 进入容器终端
- --restart 设置重启策略

### docker ps

查看运行的容器

<img src="https://cdn.kmoon.fun/2026/2026-06-20T08-51-14-369Z.png" alt="" width=500/>

- -a 查看所有的容器，包括运行的、停止的。

### docker stop/start

停止/启动容器

## Docker 技术原理

Docker 利用了 Linux 内核的两大原生功能实现容器化：

### Cgroups

用来限制和隔离进程的资源使用，可以为每个容器设定 CPU、内存、网络带宽等资源的使用上限，确保了一个容器的资源消耗不会影响到宿主机和其他正在运行的容器。

### Namespaces

用来隔离进程的资源视图，使得容器只能看到自己内部的进程 ID、网络资源、文件目录，而看不到宿主机的，所以容器本质还是一个特殊的进程。

## Dockerfile

制作镜像的图纸

docker build

## Docker 网络模式

### Bridge 桥接模式

默认是 Bridge 桥接模式

### Host 模式

