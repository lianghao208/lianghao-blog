---
title: "Kubernetes Scheduler Framework 插件化后的应用和思考"
date: 2020-11-13T14:24:26+08:00
draft: false
tags:
    - Kubernetes
    - Scheduler
---

## 一、前言
关注 Kubernetes Scheduler SIG（Special Interest Group）的朋友应该了解，在近期发布的 Kubernetes
1.19 版本中， Scheduler Framework 替代了原有 Schduler 的工作模式，正式将调度器以插件化的形式提供给了用户。相比较于旧版本的调度器”四件套“：Predicate、Priority、Bind、Preemption。新版本的调度器框架更加灵活，共引入了11个扩展点，用户可随意组装、扩展调度插件中调度算法执行的先后顺序。
参考官方的详细设计思路：[624-scheduling-framework](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework)
## 二、插件化后带来的影响
在旧版本中的默认调度器 default-scheduler 实现中，如果我们想要修改和配置该调度器的调度算法顺序和权重，我们往往需要修改 scheduler 的配置文件：

```yaml
apiVersion: v1
kind: Policy
predicate: # 定义 predicate 算法
- name: NoVolumeZoneConflict
- name: MaxEBSVolumeCount
- ...
priorities: # 定义 priorities 算法和权重
- name: SelecttorSpreadPriority
  weight: 1
- name: InterPodAffinityPriority
  weight: 1
- ...
extenders: # 定义扩展调度器服务
- urlPrefix: "http://xxxx/xxx"
  filterVerb: filter
  prioritizeVerb: prioritize
  enableHttps: false
  nodeCacheCapable: false
  ignorable: true
```
这样的配置方式并不够灵活，调度器执行调度算法的逻辑严格按照 predicate -- priorities -- extenders 的顺序执行，我们无法在调度算法和算法之间添加自定义的逻辑。我们在扩展自定义调度逻辑时，自定义的调度算法也只能在 predicate 和 priorities 之后执行，因此这样的调度器配置方式和扩展机制已不能满足我们的需求，Kuberntes 官方也慢慢在将这种调度器实现方式淘汰，取而代之的是插件化的 Scheduler Framework。
与旧版本的调度器配置方式不同，新版本的调度器在原有的”四件套“上增加到了11个扩展插件：
- **Queue sort**
    对准备调度的 Pod 进行优先级排序
- **PreFilter**
    Pod 调度前的条件检查，如果检查不通过，直接结束本调度周期（过滤带有某些标签、annotation的pod）
- **Filter**
    过滤掉不符合当前 Pod 运行条件的Node（相当于旧版本的 predicate）
- **PostFilter**
    Filter插件执行完后执行（一般用于 Pod 抢占逻辑的处理）
- **PreScore**
    打分前的状态处理
- **Scoring**
    对节点进行打分（相当于旧版本的 priorities）
- **Reserve**
- **Permit**
    Pod 绑定之前的准入控制
- **PreBind**
    绑定 Pod 之前的逻辑，如：先预挂载共享存储，查看是否正常挂载
- **Bind**
    节点和 Pod 绑定
- **PostBind**
    Pod绑定成功后的资源清理逻辑

相比较于旧版调度器的基于 webhook 的扩展机制（Scheduler Extender），Scheduler Framework 使用的是将自定义的调度器进行二进制编译，替换掉原有的调度器的方式。这样做主要是为了解决旧版调度器扩展机制存在的一些问题：
- **调度算法执行时机不灵活**
    扩展的predicate、priority算法只能在默认调度器的调度算法后执行
- **序列化、反序列化慢**
    扩展调度器使用HTTP+JSON的形式序列化、反序列化
- **调用扩展调度器异常后当前Pod的调度任务直接被丢弃**
    需要一种通知机制来让scheduler感知到扩展调度器异常
- **无法共享默认调度器的缓存**（因为扩展调度器和默认调度器在不同的进程）
## 三、如何配置 Scheduler Framework 插件
新版本 Scheduler Framework 调度器和旧版本使用上区别不大，如果使用 kubeadm 安装集群，scheduler 默认以静态 Pod 的形式运行在集群中，如果我们需要对调度器的配置进行调整，修改调度器配置 KubeSchedulerConfiguration 即可：
```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: xxx
profiles:
- plugins:
    queueSort:
      enabled:
      - name: "*"
    preFilter:
      enabled:
      - name: "*"
    filter:
      enabled:
      - name: "*"
    preScore:
      enabled:
      - name: "*"
    score:
      enabled:
      - name: "*"
```
使用 * 号通配符表示使用该调度器所有已提供的调度器实现。在 1.19 的源码中，pkg/scheduler/framework/plugins 目录下为 Kubernetes 的调度器内置调度算法的各种实现：
![内置调度算法](https://img-blog.csdnimg.cn/20201028211701216.png#pic_center)
如需要禁用调度器的某个调度算法，只需要在 KubeSchedulerConfiguration 配置中相应的 plugin 下 disabled 掉即可。

## 四、如何二次开发并使用自定义的 Scheduler Framework 插件
当我们需要新增自定义调度器逻辑时，直接把自定义的调度器编译成二进制，打包成镜像运行在集群中，替换默认的调度器（通过修改 Pod 的 schedulerName 字段替换默认调度器），同时修改Scheduler Framework 的调度器配置。
例如我们自定义了一个叫 Coscheduling 的调度器，该调度器扩展了queueSort、preFilter、permit、reserve、postBind 插件，我们需要对它做如下配置即可：
```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: xxx
profiles:
- plugins:
    queueSort:
      enabled:
      - name: Coscheduling
      disabled:
      - name: "*"
    preFilter:
      enabled:
      - name: Coscheduling
      disabled:
      - name: "*"
    filter:
      disabled:
      - name: "*"
    preScore:
      disabled:
      - name: "*"
    score:
      disabled:
      - name: "*"
    permit:
      enabled:
      - name: Coscheduling
    reserve:
      enabled:
      - name: Coscheduling
    postBind:
      enabled:
      - name: Coscheduling
  pluginConfig:
  - name: Coscheduling
    args:
      permitWaitingTimeSeconds: 10
      kubeConfigPath: "%s"
```
那么二次开发自定义调度器的流程大概是怎样的呢？官方为每个插件提供了接口，我们只要在自己的调度器中实现这些接口即可：

```go
// pkg/scheduler/framework/v1alpha1/interface.go
type QueueSortPlugin interface {
	Plugin
	Less(*QueuedPodInfo, *QueuedPodInfo) bool
}

type PreFilterExtensions interface {
	AddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
	RemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToRemove *v1.Pod, nodeInfo *NodeInfo) *Status
}

type PreFilterPlugin interface {
	Plugin
	PreFilter(ctx context.Context, state *CycleState, p *v1.Pod) *Status
	PreFilterExtensions() PreFilterExtensions
}

type FilterPlugin interface {
	Plugin
	Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
}

type PostFilterPlugin interface {
	Plugin
	PostFilter(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
}

type PreScorePlugin interface {
	Plugin
	PreScore(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
}

type ScoreExtensions interface {
	NormalizeScore(ctx context.Context, state *CycleState, p *v1.Pod, scores NodeScoreList) *Status
}

type ScorePlugin interface {
	Plugin
	Score(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (int64, *Status)
	ScoreExtensions() ScoreExtensions
}

type ReservePlugin interface {
	Plugin
	Reserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
	Unreserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
}

type PreBindPlugin interface {
	Plugin
	PreBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
}

type PostBindPlugin interface {
	Plugin
	PostBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
}

type PermitPlugin interface {
	Plugin
	Permit(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (*Status, time.Duration)
}

type BindPlugin interface {
	Plugin
	Bind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
}
```
具体的实现代码可参考 Kubernetes 官方提供的样例： [scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)
## 五、总结
从 Kubernetes 官方提出重构 Scheduler 的 Proposal，到新版 Scheduler Framework 正式上线的时间跨度大概有一年多，得益于 Kubernetes 社区的高活跃度和高参与度，重构工作能够顺利完成。然而，即便调度器框架已完成重构，但现有的调度器算法其实并不完善，还不能完全满足集群容器编排的各种调度需求，如：基于真实负载的调度算法、基于 GPU 的调度算法等，未来还有待完善。
说到调度器，不得不提起 Kubernetes 的另一个开源项目：Descheduler（反调度器），其功能在于动态调整**已完成调度**的 Pod 的调度结果，下次有机会再详细说说其实现原理和应用场景。