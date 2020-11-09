---
title: "K8S-APIServer日志报 has no resources 的分析"
date: 2020-03-22T14:24:26+08:00
draft: false
---

# 一、资源超卖问题分析
在生产环境中，kubernetes集群的计算节点上运行着许许多多的Pod，分别跑着各种业务容器，我们通常用Deployment、DeamonSet、StatefulSet等资源对象去控制Pod的增删改。因此，开发或运维往往需要配置这些资源对象的Containers字段中业务容器的CPU和内存的资源配额：requests和limit

 - requests：节点调度pod需要的资源，每次成功调度则将节点的Allocatable属性值（可分配资源）重新计算，
 新的Allocatable值 = 旧的Allocatable值 - 设置的requests值
 - limit：节点中运行pod能够获得的最大资源，当cpu

我们不难发现，当requests字段设置太大的时候，pod实际使用的资源却很小，导致计算节点的Allocatable值很快就被消耗完，节点的资源利用率会变得很低。
![节点资源占用情况](https://img-blog.csdnimg.cn/20200322114901919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
上图中最大的蓝色框（allocatable）为计算节点可分配资源，橙色框（requests）为用户配置的requests属性，红色框（current）为业务容器实际使用的资源。因此节点的资源利用率为 current / allocatable。而由于requests设置太大，占满了allocatable，导致新的pod无法被调度到这个节点，就会出现**节点实际资源占用很低，却因为allocatable太低导致pod无法调度到该节点**的现象。
因此我们能否通过动态调整allocatable的值来让计算节点的可分配资源变得"虚高"，骗过k8s的调度器，让它以为该节点可分配资源很大，让尽可能多的pod调度到该节点上呢？
![动态调整allocatable属性](https://img-blog.csdnimg.cn/20200322120030803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
上图通过将allocatable值扩大（fake allcatable），让更多的pod调度到了改节点，节点的资源利用率 current / allocatable 就变大了。

```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    ...
  creationTimestamp: ...
  labels:
    ...
  name: k8s-master.com
  resourceVersion: "7564956"
  selfLink: /api/v1/nodes/k8s-master.com
  uid: ...
spec:
  podCIDR: 10.244.0.0/24
status:
  addresses:
  - address: 172.16.0.2
    type: InternalIP
  - address: k8s-master.com
    type: Hostname
  allocatable: ## 这就是需要动态修改的字段！！！
    cpu: "1"
    ephemeral-storage: "47438335103"
    hugepages-2Mi: "0"
    memory: 3778260Ki
    pods: "110"
  capacity:
    cpu: "1"
    ephemeral-storage: 51473888Ki
    hugepages-2Mi: "0"
    memory: 3880660Ki
    pods: "110"
  conditions:
    ...
  daemonEndpoints:
    ...
  images: 
    ...
```

# 二、实现资源超卖的思路
实现资源超卖的关键在于动态修改节点Node对象的allocatable字段值，而我们看到allocatable字段属于Status字段，显然不能直接通过kubectl edit命令来直接修改。因为Status字段和Spec字段不同，Spec是用户设置的期望数据，而Status是实际数据（Node节点通过不断向apiServer发送心跳来更新自己的实时状态，最终存在etcd中）。那么我们要怎么去修改Stauts字段呢？
首先，要修改k8s中任何资源对象的Status值，k8s官方提供了一套RESTful API：[https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13)
可以通过patch或者put方法来调用k8s的RESTful API，实现Stauts字段的修改。（这里是通过ApiServer去修改etcd中保存的Status字段的值）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200322133615880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
但是，Node资源对象比较特殊，计算节点会不断给ApiServer发送心跳（默认每隔10s发一次），将带有Status字段的真实信息发送给ApiServer，并更新到etcd中。也就是无论你怎么通过patch/put方法去修改Node的Status字段，计算节点都会定时通过发送心跳将真实的Status数据覆盖你修改的数据，也就是说我们无法通过直接调用RESTful API修改Node对象中的Status数据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200322134459160.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
那我们是否可以直接监听这个计算节点的心跳数据，通过修改心跳数据中的Status字段中的allocatable值，从而实现资源超卖呢？
# 三、MutatingAdmissionWebhook特性
答案是肯定的，k8s在ApiServer中就提供了Admission Controller（准入控制器）的机制，其中包括了MutatingAdmissionWebhook，通过这个webhook，所有和集群中所有和ApiSever交互的请求都被发送到一个指定的接口中，我们只要提供一个这样的接口，就可以获取到Node往ApiServer发送心跳的Staus数据了。然后将这个数据进行我们的自定义修改，再往后传给etcd，就能让etcd以为我们修改过的Status数据就是节点的真实Status，最终实现资源的超卖。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200322144828984.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)

> MutatingAdmissionWebhook作为kubernetes的ApiServer中Admission Controller的一部分，提供了非常灵活的扩展机制，通过配置MutatingWebhookConfiguration对象，理论上可以监听并修改任何经过ApiServer处理的请求

# 四、MutatingWebhookConfiguration对象简介

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  creationTimestamp: null
  name: mutating-webhook-oversale
webhooks:
- clientConfig:
    caBundle: ...
    service:
      name: webhook-oversale-service
      namespace: oversale
      path: /mutate
  failurePolicy: Ignore
  name: oversale
  rules:
  - apiGroups:
    - *
    apiVersions:
    - v1
    operations:
    - UPDATE
    resources:
    - nodes/status

```
MutatingWebhookConfiguration是kubernetes的一个官方的资源提供的对象，下面对该对象的字段做一些简单的说明：

 - clientConfig.caBundle：apiServer访问我们自定义的webhook服务时需要的加密认证数据
 - clientConfig.service：apiServer访问我们自定义的webhook服务的Service相关信息（包括具体接口信息）
 - failurePolicy：当apiServer调用我们自定义的webhook服务异常时，采取的策略（Ignore：忽略异常继续处理，Fail：直接失败退出不继续处理）
 - rules.operations：监听apiServer的操作类型，样例中，只有符合UPDATE类型的apiServer调用才会交给我们自定义的webhook服务处理。
 - rules.resources：监听apiServer的资源和子资源类型，样例中，只有符合nodes的status字段资源类型的apiServer调用才会交给我们自定义的webhook服务处理。

结合rules.operations和rules.resources的属性，我们可以知道样例中的**MutatingWebhookConfiguration监听了集群中nodes资源的status数据向apiServer提交的更新操作**（就是我们前面提到的**心跳信息**），并且将所有的心跳信息发给了名为webhook-oversale-service的Service下的/mutate接口处理，这个接口就是我们自定义的webhook服务提供的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200322150846693.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
上图中的Pod跑着的容器就是我们自定义的webhook服务，一个自定义webhook服务样例供参考：[admission-webhook-oversale-sample](https://gitee.com/lianghaocs/admission-webhook-oversale-sample)
# 五、源码分析
未完待续
# 六、资源超卖算法实践
未完待续
# 七、参考资料
[kubernetes RESTful API docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13)
[腾讯自研业务上云：优化 Kubernetes 集群负载的技术方案探讨](https://www.infoq.cn/article/wyjT7HApETsiEAMoiL7Z)
[api-conventions：spec-and-status](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)