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
### OCI（开放容器标准）
本质上是**一篇文档**，规定了2点：
- 容器镜像要长啥样，即 ImageSpec。里面的大致规定就是你这个东西需要是一个压缩了的文件夹，文件夹里以 xxx 结构放 xxx 文件；
- 容器要需要能接收哪些指令，这些指令的行为是什么，即 RuntimeSpec。这里面的大致内容就是“容器”要能够执行 “create”，“start”，“stop”，“delete” 这些命令，并且行为要规范。

Docker 把 libcontainer 封装了一下，变成 runC 捐献出来作为 OCI 的参考实现。
Docker安装成功后，容器引擎运行着两个守护进程分别是dockerd和docker-containerd。当有新的容器被创建时，Docker 会创建新的进程 docker-containerd-shim，从而控制底层的 runc。
### CRI（容器运行时接口）
本质上是**一组 gRPC 接口**：
- 操作容器的接口，包括创建、启停容器等
- 操作镜像的接口，包括拉取、删除镜像等
- 操作 PodSandbox（容器沙箱环境）接口

由于历史原因，Docker 最开始和 CRI 是不兼容的，Kubelet 如果需要通过 CRI 控制 Docker，则需要一个转换器来使其适配 CRI，这就是 docker shim（垫片），shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上。
因此，docker-shim 的作用是为了在满足 CRI 规范的条件下控制 Docker，而 docker-containerd-shim 则是为了让 containerd 满足 OCI 规范（Open Container Initiative，开放容器标准）。
## 三、为什么 dockershim 被弃用？
由于 Kubelet 通过调用 Docker 从而操作容器的实现方式太过复杂，Kubernetes 后来推出了 CRI-O 的概念逐渐将 Docker 边缘化。CRI-O 就是一个兼容 CRI 和 OCI 的 Kubernetes 专用容器运行时实现。
![CRI-O](https://img-blog.csdnimg.cn/20201206200322806.png)
早在2020年的7月份，Kubernetes 社区就出现了移除 dockershim 的提案：[https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1985-remove-dockershim](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1985-remove-dockershim)
在Kubernetes 1.20 中使用 Docker 作为运行时，在kubelet启动时打印一个警告日志。（目前并没有移除，只是弃用，官方表示它不会在 Kubernetes 1.22 之前被删除，这意味着最早不使用 dockershim 的版本将是2021年底的1.23）
## 四、对运维和开发人员的影响
对开发人员而言，构建镜像时仍然可以使用 Docker，因为其满足 OCI 标准，构建出来的镜像自然满足 OCI。无论你使用什么工具构建它，任何符合 OCI 标准的镜像在 Kubernetes 看来都是一样的。
而对于运维人员而言，想要操作运行时的容器，crictl 看似一个不错的选择，当然这也可能带来一定的时间成本。