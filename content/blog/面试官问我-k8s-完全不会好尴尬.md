+++
title = "面试官问我 K8s 完全不会好尴尬"
date = "2026-04-05T21:45:40+08:00"
tags = ["Kubernetes","k8s"]
+++

> Reference: 
> - [小白debug](https://www.bilibili.com/video/BV1Du4m137pK/)
> - [什么是 Kubernetes 集群？](https://aws.amazon.com/cn/what-is/kubernetes-cluster/)

## 背景

上次面试腾讯云智暑期实习一面，面试官突然问我有没有了解过 k8s，我只好说不太熟悉，并说熟悉 docker，但是面试官并没有继续追问，显然是对不熟悉 k8s 不满意，所以这次我们补齐一下 k8s 的相关知识。

## 什么是 k8s

k8s 原名 Kubernetes 是一组运行容器化应用程序的计算节点或工作机，是 Google 开源的一个工具。因为单词太长，将中间 8 个字母简化后统称 k8s。k8s 其实是应用服务和服务器之间的一层中间层，定位是可以帮助我们更方便的部署和运维应用服务，可以同时协调和管理多个应用服务。

传统的应用服务部署方式，需要我们先将代码上传到不同的服务器上，再在服务器配置所需环境，再配置 nginx 网关，最后启动服务。后续不管是扩容、还是更新都需要重新手动登录到服务器操作，非常容易出错，而且非常耗时间。

### k8s 架构

k8s 的整体架构如下图所示，主要包括 control panel 控制平面和工作节点 Node 两部分，其中控制平面负责控制和管理各个 Node，Node 部分才是负责真正部署运行应用服务的，就像老板和打工人一样。

<img src="https://cdn.kmoon.fun/2026/2026-06-19T09-33-14-811Z.png" alt="" width=500/>

控制平面内部组件包括：

- API Server：提供用户操作服务器的 API 接口
- Scheduler：资源协调调度器
- Controller Mgr：控制器管理器，控制 Node 的创建和关闭
- ETCD：存储层，保存相关过程数据

<img src="https://cdn.kmoon.fun/2026/2026-06-19T09-47-12-445Z.png" alt="" width=300/>

Node 内部组件包括：

- Kubelet：负责管理和监控 Pod
- Kube Proxy：负责 Node 的网络通信功能，转发外部请求到 Pod 中
- Pod：由多个 contain 组成，一般为应用服务、日志收集器、监控采集器 3 个 Container 组成
- Container runtime：容器运行时组件，负责下载和部署镜像

<img src="https://cdn.kmoon.fun/2026/2026-06-19T09-53-38-357Z.png" alt="" width=300/>

Pod 是 Kubernetes 中的最小调度单元，一个 Pod 可由多个容器（Container）组成，单个节点（Node）上会运行多个 Pod。一套控制面搭配若干 Node 共同构成完整的 Kubernetes 集群（Cluster）。

一般生产环境会部署多个集群，比如生产环境集群、测试环境集群。同时为里暴露集群内部的服务给外部用户使用，通常会部署一个入口控制器 ingress。

## Kubectl

k8s 为我们准备的一个命令行工具，可以通过命令去调用 API Server 提供的 API 服务。

<img src="https://cdn.kmoon.fun/2026/2026-06-19T12-21-39-892Z.png" alt="" width=500/>

