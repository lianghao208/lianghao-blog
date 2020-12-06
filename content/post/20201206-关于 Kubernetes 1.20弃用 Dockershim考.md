---
title: "关于 Kubernetes 1.20弃用 Dockershim"
date: 2020-12-06T14:24:26+08:00
draft: false
tags:
    - Kubernetes
    - Docker
---

## 一、背景
近期，Kubernetes 1.20 版本发布，查看该版本的 [CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation)，发现 Kubernetes 在1.20版本之后将弃用 Docker 作为容器运行时。
我们知道，Docker 是容器运行时接口（CRI）的一种实现。而 CRI 接口的规范最早在 Kubernetes 1.5 版本中引入：[https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)。其目的是为了适配多种容器运行时，让所有具有管理容器生命周期的系统实现这一统一标准接口（容器查看、创建、删除、更新等），从而通过上层的 Kubernetes 去统一管理。
而在 Kubernetes 1.20 版本中，Docker 作为底层的 CRI 实现正在被弃用，取而代之的是使用为Kubernetes 自己创建的 CRI 实现。当然，这不意味着 Docker 不再适用于 Kubernetes，Docker 仍然是构建容器的有用工具，运行 Docker build 产生的镜像仍然可以在 Kubernetes 集群中运行。

## 二、docker-shim、docker、containerd、containerd-shim、runc之间的关系
![Kubelet 兼容 Docker 的架构](https://img-blog.csdnimg.cn/20201206162401902.png)
Docker安装成功后，容器引擎运行着两个守护进程分别是dockerd和docker-containerd。当有新的容器被创建时，Docker 会创建新的进程 docker-containerd-shim，从而控制底层的 runc。
由于历史原因，Docker 最开始和 CRI 是不兼容的，Kubelet 如果需要通过 CRI 控制 Docker，则需要一个转换器来使其适配 CRI，这就是 docker-shim（垫片）。
因此，docker-shim 的作用是为了在满足 CRI 规范的条件下控制 Docker，而 docker-containerd-shim 则是为了让 containerd 满足 OCI 规范（Open Container Initiative，开放容器标准）。