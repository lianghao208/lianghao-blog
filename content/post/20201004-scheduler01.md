---
title: "K8S调度器--kube-scheduler架构设计和启动流程源码分析"
date: 2020-10-04T14:24:26+08:00
draft: false
---

@[TOC]
## 一、kube-scheduler架构设计
![架构设计](https://img-blog.csdnimg.cn/20201013220956282.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70#pic_center)
调度器的核心功能是为Pod找到最适合的节点运行。对于小规模集群，每个调度周期会遍历集群中的所有节点，找到最合适节点进行调度。而对于大规模集群，每个调度周期只会遍历集群中的部分节点，在这部分节点中找到最合适的节点进行调度。
整个调度流程主要分为预选、优选和绑定三个节点。预选阶段首先过滤掉不符合条件的节点，优选阶段主要对预选阶段筛选后的节点进行打分，绑定阶段将分数最高的节点和pod进行绑定，完成调度。
**此源码分析针对 Kubernetes V1.18.10 版本**
## 二、kube-scheduler组件启动流程
![组件启动流程](https://img-blog.csdnimg.cn/20201016232658227.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70#pic_center)
### 2.1 内置调度算法注册
在 createFromProvider 函数中调用 algorithmprovider.NewRegistry() 注册调度器算法插件：

```go
//pkg/scheduler/factory.go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		prof.Plugins = plugins
	}
	return c.create()
}
```
algorithmprovider.NewRegistry() 函数调用了 getDefaultConfig() 获取默认调度算法，将其注册。
```go
//pkg/scheduler/algorithmprovider/registry.go
func getDefaultConfig() *schedulerapi.Plugins {
	return &schedulerapi.Plugins{
		QueueSort: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: queuesort.Name},
			},
		},
		PreFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.FitName},
				{Name: nodeports.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
				{Name: volumebinding.Name},
			},
		},
		Filter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: nodeunschedulable.Name},
				{Name: noderesources.FitName},
				{Name: nodename.Name},
				{Name: nodeports.Name},
				{Name: nodeaffinity.Name},
				{Name: volumerestrictions.Name},
				{Name: tainttoleration.Name},
				{Name: nodevolumelimits.EBSName},
				{Name: nodevolumelimits.GCEPDName},
				{Name: nodevolumelimits.CSIName},
				{Name: nodevolumelimits.AzureDiskName},
				{Name: volumebinding.Name},
				{Name: volumezone.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
			},
		},
		PostFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultpreemption.Name},
			},
		},
		PreScore: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: interpodaffinity.Name},
				{Name: podtopologyspread.Name},
				{Name: tainttoleration.Name},
			},
		},
		Score: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.BalancedAllocationName, Weight: 1},
				{Name: imagelocality.Name, Weight: 1},
				{Name: interpodaffinity.Name, Weight: 1},
				{Name: noderesources.LeastAllocatedName, Weight: 1},
				{Name: nodeaffinity.Name, Weight: 1},
				{Name: nodepreferavoidpods.Name, Weight: 10000},
				// Weight is doubled because:
				// - This is a score coming from user preference.
				// - It makes its signal comparable to NodeResourcesLeastAllocated.
				{Name: podtopologyspread.Name, Weight: 2},
				{Name: tainttoleration.Name, Weight: 1},
			},
		},
		Reserve: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		PreBind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		Bind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultbinder.Name},
			},
		},
	}
}
```
Filter 中设置的为 Predicate 预选算法插件列表，Score 中设置的为 Priority 优选算法插件列表。将所有插件初始化完成后，注册放入 schedulerapi.Plugins{} 调度器插件列表中。
### 2.2 Cobra 命令行参数解析
Cobra 是 Kubernetes、Docker 等系统中使用的命令行解析工具，kube-scheduler 组件也使用了 Cobra 做命令行解析。

```go
//cmd/kube-scheduler/app/server.go
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
	//初始化命令行默认配置
	opts, err := options.NewOptions()
	...
	cmd := &cobra.Command{
		Use: "kube-scheduler",
		Long: ...
		Run: func(cmd *cobra.Command, args []string) {
		    // runCommand 函数对命令行的命令、参数进行解析、校验
			if err := runCommand(cmd, opts, registryOptions...); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}
	...
	return cmd
}
// runCommand 函数对命令行的命令、参数进行解析、校验
func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
	verflag.PrintAndExitIfRequested()
	cliflag.PrintFlags(cmd.Flags())

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	cc, sched, err := Setup(ctx, opts, registryOptions...)
	...
	//运行 Run 函数，启动 kube-scheduler 组件
	return Run(ctx, cc, sched)
}
```
### 2.3 实例化 Scheduler 对象
Scheduler 对象实例化主要实例化了资源对象的 Informer、调度算法和资源对象事件的监控。
```go
//cmd/kube-scheduler/app/server.go
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	...
	// Create the scheduler.
	// 实例化 scheduler 对象
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
	)
	...
	return &cc, sched, nil
}
```
实例化 scheduler 对象 New 函数的具体执行逻辑
```go
//pkg/scheduler/scheduler.go
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	...
	// 1、实例化所有 Informer
	configurator := &Configurator{
		client:                   client,
		recorderFactory:          recorderFactory,
		informerFactory:          informerFactory,
		schedulerCache:           schedulerCache,
		StopEverything:           stopEverything,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		podInitialBackoffSeconds: options.podInitialBackoffSeconds,
		podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
		registry:                 registry,
		nodeInfoSnapshot:         snapshot,
		extenders:                options.extenders,
		frameworkCapturer:        options.frameworkCapturer,
	}
	...
	var sched *Scheduler
	source := options.schedulerAlgorithmSource
	switch {
	case source.Provider != nil:
		// 2、实例化调度算法
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
		...
	}
	...
	// 3、为所有 Informer 对象添加对资源事件的监控
	addAllEventHandlers(sched, informerFactory)
	return sched, nil
}
```
### 2.4 运行 EventBroadcaster 事件管理器
Event（事件）作为 Kubernetes 的一种资源对象，用于描述集群产生的事件。如 kube-scheduler 组件在调度过程中，对某个 Pod 的调度做了什么处理、为什么需要对某些 Pod 做驱逐等决策都通过向 apiserver 请求创建 Event 记录下来，新创建的 Event 默认保留1个小时。

```go
//cmd/kube-scheduler/app/server.go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	...
	// 运行 EventBroadcaster
	cc.EventBroadcaster.StartRecordingToSink(ctx.Done())
	...
	return fmt.Errorf("finished without leader elect")
}

```

### 2.5 运行 HTTP 或 HTTPS 服务
kube-scheduler 中提供了三个 HTTP 服务：健康检测、指标监控、pprof 性能分析。分别对应 /healthz、/metrics、/debug/pprof 三个接口。
```go
//cmd/kube-scheduler/app/server.go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	...
	// 启动 /healthz 健康检测服务
	if cc.InsecureServing != nil {
		separateMetrics := cc.InsecureMetricsServing != nil
		handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, separateMetrics, checks...), nil, nil)
		if err := cc.InsecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start healthz server: %v", err)
		}
	}
	// 启动 /metrics 指标监控
	if cc.InsecureMetricsServing != nil {
		handler := buildHandlerChain(newMetricsHandler(&cc.ComponentConfig), nil, nil)
		if err := cc.InsecureMetricsServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start metrics server: %v", err)
		}
	}
	...
	return fmt.Errorf("finished without leader elect")
}
```
### 2.6 运行 Informer 同步资源
在 2.3 中已经实例化好了所有的 Informer 对象，在这一步需要将它们运行起来。

```go
//cmd/kube-scheduler/app/server.go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	...
	// 运行所有 Informer
	cc.InformerFactory.Start(ctx.Done())
	// 等待所有 Informer 缓存同步完成
	cc.InformerFactory.WaitForCacheSync(ctx.Done())
	...
	return fmt.Errorf("finished without leader elect")
}
```

### 2.7 Leader 选举实例化
Kubernetes 中 controller-manager 和 kube-scheduler 组件都有 Leader 选举机制，通过选主来实现分布式系统的高可用。其实现原理是通过定义回调函数 leaderelection.LeaderCallbacks 中的 OnStartedLeading 和 OnStoppedLeading 来回调通知当前节点的 kube-scheduler 组件是否竞争 Leader 成功。OnStartedLeading 回调表示通过选举成为了 Leader，因此调用 sched.Run 启动调度器。而 OnStoppedLeading 回调表示选举成为 Leader 失败、被别的节点抢占了 Leader，会打出 Fatal 日志直接退出调度器进程。
```go
//cmd/kube-scheduler/app/server.go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	
	// 判断是否开启选举
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: sched.Run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		// 开启新一轮选举
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}
	...
	return fmt.Errorf("finished without leader elect")
}
```

### 2.8 sched.Run 运行调度器
```go
//cmd/kube-scheduler/app/server.go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	...
	// 运行调度器
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```
下面是 sched.Run(ctx) 的主要逻辑，sched.scheduleOne 执行调度器的调度逻辑，通过wait.UntilWithContext 定时器执行（第三个参数为0，表示 sched.scheduleOne 函数执行完后立马又再次执行），定时调用 sched.scheduleOne 函数，直到主协程通过 cancel 通知 ctx 上下文结束。
```go
//pkg/scheduler/scheduler.go
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```
