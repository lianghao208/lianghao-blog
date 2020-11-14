---
title: "Kubernetes 的HPA机制"
date: 2020-03-13T14:24:26+08:00
draft: false
tags:
    - Kubernetes
    - HPA
---

# 一、HPA简介
HPA（Horizontal Pod Autoscaler）Pod自动弹性伸缩，K8S通过对Pod中运行的容器各项指标（CPU占用、内存占用、网络请求量）的检测，实现对Pod实例个数的动态新增和减少。

早期的kubernetes版本，只支持CPU指标的检测，因为它是通过kubernetes自带的监控系统heapster实现的。

到了kubernetes 1.8版本后，heapster已经弃用，资源指标主要通过metrics api获取，这时能支持检测的指标就变多了（CPU、内存等**核心指标**和qps等**自定义指标**）
# 二、HPA设置
HPA是一种资源对象，通过yaml进行配置：

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: 10k
```
**minReplicas：** 最小pod实例数

**maxReplicas：** 最大pod实例数

**metrics：** 用于计算所需的Pod副本数量的指标列表

**resource：** 核心指标，包含cpu和内存两种（被弹性伸缩的pod对象中容器的requests和limits中定义的指标。）

**object：** k8s内置对象的特定指标（需自己实现适配器）

**pods：** 应用被弹性伸缩的pod对象的特定指标（例如，每个pod每秒处理的事务数）（需自己实现适配器）

**external：** 非k8s内置对象的自定义指标（需自己实现适配器）

# 三、HPA获取自定义指标（Custom Metrics）的底层实现（基于Prometheus）
Kubernetes是借助Agrregator APIServer扩展机制来实现Custom Metrics。Custom Metrics APIServer是一个提供查询Metrics指标的API服务（Prometheus的一个适配器），这个服务启动后，kubernetes会暴露一个叫custom.metrics.k8s.io的API，当请求这个URL时，请求通过Custom Metics APIServer去Prometheus里面去查询对应的指标，然后将查询结果按照特定格式返回。

具体步骤可参考：https://github.com/resouer/kubeadm-workshop
（创建Custom Metrics APIServer、Prometheus以及HPA样例的yaml文件在demos/monitoring目录下）

HPA样例配置：

```
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-metrics-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-metrics-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      target:
        kind: Service
        name: sample-metrics-app
      metricName: http_requests
      targetValue: 100
```


当配置好HPA后，HPA会向Custom Metrics APIServer发送https请求：

```
https://<apiserver_ip>/apis/custom-metrics.metrics.k8s.io/v1beta1/namespaces/default/services/sample-metrics-app/http_requests
```
可以从上面的https请求URL路径中得知，这是向 **default** 这个 namespaces 下的名为 **sample-metrics-app** 的 service 发送获取 **http_requests** 这个指标的请求。

Custom Metrics APIServer收到 **http_requests** 查询请求后，向Prometheus发送查询请求查询 **http_requests_total** 的值（总请求次数），Custom Metics APIServer再将结果计算成 **http_requests** （单位时间请求率）返回，实现HPA对性能指标的获取，从而进行弹性伸缩操作。

指标获取流程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200225221949880.PNG)

如何自定义Adapter的指标：https://github.com/DirectXMan12/k8s-prometheus-adapter

Helm的方式自定义Adapter的指标：
https://github.com/helm/charts/blob/master/stable/prometheus-adapter/README.md
