---
title: "Client-go源码分析（四）WorkQueue"
date: 2020-08-21T14:24:26+08:00
draft: false
---

## 一、WorkQueue 简介
在 Informer 中，Delta FIFO 队列触发 Add、Update、Delete 回调，在回调方法中，将需要处理的资源对象变化事件的 key 放入 WorkQueue 工作队列中。等待 Control Loop 控制循环从工作队列中取出，再通过 Lister 从 Indexer 本地缓存中取出完整的资源对象，做处理。
![Informer 工作流程架构图](https://img-blog.csdnimg.cn/20200810221241261.png)
图片来源：极客时间 --《深入浅出 Kubernetes》
WorkQueue 的主要功能在于标记和去重，并支持以下特性：
- 有序：按照添加顺序处理元素（item）
- 去重：相同元素在同一时间不会被重复处理，例如一个元素在处理前被添加了多次，它只会被处理一次。
- 并发性：多生产者和多消费者
- 标记机制：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
- 通知机制：ShutDown方法通过信号量通知队列不再接收新的元素，并通知metric goroutine退出。
- 延迟：支持延迟队列，延迟一段时间后再将元素存入队列。
- 限速：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队（Reenqueued）的次数。
- Metric：支持metric监控指标，可用于Prometheus监控。
## 二、WorkQueue 结合 Informer 的完整使用案例
Kubernetes 官方给出了使用 Informer 机制实现 controller 的样例：

```go
// staging/src/k8s.io/sample-controller/controller.go
// 1、定义 Controller 结构体
type Controller struct {
	// 官方的 Clientset 客户端获取内置资源对象
	kubeclientset kubernetes.Interface
	// 自定义 Clientset 客户端获取自定义资源对象
	sampleclientset clientset.Interface

	deploymentsLister appslisters.DeploymentLister
	deploymentsSynced cache.InformerSynced
	foosLister        listers.FooLister
	foosSynced        cache.InformerSynced
	// 使用限流队列实现的 WorkQueue
	workqueue workqueue.RateLimitingInterface
	recorder record.EventRecorder
}
// 2、初始化 controller 对象
func NewController(
	kubeclientset kubernetes.Interface,
	sampleclientset clientset.Interface,
	deploymentInformer appsinformers.DeploymentInformer,
	fooInformer informers.FooInformer) *Controller {
	...

	controller := &Controller{
		kubeclientset:     kubeclientset,
		sampleclientset:   sampleclientset,
		deploymentsLister: deploymentInformer.Lister(),
		deploymentsSynced: deploymentInformer.Informer().HasSynced,
		foosLister:        fooInformer.Lister(),
		foosSynced:        fooInformer.Informer().HasSynced,
		workqueue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Foos"),
		recorder:          recorder,
	}

	// 为自定义资源 Foo 初始化 Informer，设置事件回调函数
	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		// 如果从 FIFO 队列中取到 Foo 对象的 Add 事件，则放入 workQueue 中 
		AddFunc: controller.enqueueFoo,
		// 如果从 FIFO 队列中取到 Foo 对象的 Update 事件，则将新的对象放入 workQueue 中
		UpdateFunc: func(old, new interface{}) {
			controller.enqueueFoo(new)
		},
	})
	...
	return controller
}

// 3、workQueue 的入队逻辑，将对象的 key 放入 WorkQueue 中
func (c *Controller) enqueueFoo(obj interface{}) {
	var key string
	var err error
	if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
		runtime.HandleError(err)
		return
	}
	c.workqueue.AddRateLimited(key)
}
// 4、开启 Control Loop 逻辑
func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
	...
	// 开启协程跑 Control Loop
	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}
	...
	return nil
}
// 5、不断循环调用 processNextWorkItem 从 workQueu 队列中取对象处理
func (c *Controller) runWorker() {
	for c.processNextWorkItem() {
	}
}
func (c *Controller) processNextWorkItem() bool {
	// woekQueu 队列中获取队列头的元素
	obj, shutdown := c.workqueue.Get()
	...
	// We wrap this block in a func so we can defer c.workqueue.Done.
	err := func(obj interface{}) error {
		defer c.workqueue.Done(obj)
		var key string
		var ok bool
		if key, ok = obj.(string); !ok {
			// 如果出现异常则将对象出列，否则队列中会一直存在异常的元素
			c.workqueue.Forget(obj)
			runtime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
			return nil
		}
		// 调用 syncHandler 实现 controller 真正的处理逻辑
		if err := c.syncHandler(key); err != nil {
			return fmt.Errorf("error syncing '%s': %s", key, err.Error())
		}
		// 处理完成后将元素推出队列
		c.workqueue.Forget(obj)
		glog.Infof("Successfully synced '%s'", key)
		return nil
	}(obj)
	...
	return true
}
// 6、controller 的核心处理逻辑
func (c *Controller) syncHandler(key string) error {
	// 从 key 分离出资源对象的 namespace 和名字
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	...
	// 通过 Lister 从 Indexer 中获取本地缓存的完整对象
	foo, err := c.foosLister.Foos(namespace).Get(name)
	...
	// 下面就是为获取到的对象写自定义的控制逻辑
	...
	...
	return nil
}
```
WorkQueue支持3种队列：
- Interface：FIFO队列接口，先进先出队列，并支持去重机制。
- DelayingInterface：延迟队列接口，基于Interface接口封装，延迟一段时间后再将元素存入队列。
- RateLimitingInterface：限速队列接口，基于DelayingInterface接口封装，支持元素存入队列时进行速率限制。

下面介绍三种队列的差异。
## 三、FIFO 队列
FIFO 是最基本的先进先出队列，使用起来非常简单：

```go
// 初始化 FIFO 队列
workqueue:= workqueue.NewNamed("Foos")
// 入队
workqueu.Add(xxx)
// 队列头取出元素放入 processing 集合中（表示该元素正在处理）
workqueu.Get(xxx)
// 完成 processing 集合的某元素的处理（表示该元素处理完成）
workqueu.Done(xxx)
```
队列实现了 Interface 接口：
```go
type Interface interface {
	// 入队
	Add(item interface{})
	// 获取队列长度
	Len() int
	// 获取队列头的元素
	Get() (item interface{}, shutdown bool)
	// 标记队列头元素已经被处理
	Done(item interface{})
	// 关闭队列
	ShutDown()
	// 查询队列是否处于关闭状态
	ShuttingDown() bool
}
```
FIFO 队列数据结构：

```go
type Type struct {
	// 实际存储元素的切片（保证元素有序）
	queue []t
	// 用于去重，保证同一个元素被添加多次后只会被处理一次（底层 HashMap）
	dirty set
	// 用于标记一个元素是否正在被处理（底层 HashMap）
	processing set
	cond *sync.Cond
	shuttingDown bool
	metrics queueMetrics
}
```
key 为1、2、3的元素入队，当调用 Get 方法，从队列头取出1，放入 processing 集合中表示正在处理该元素，处理完1后，调用 Done 方法标记该元素已经被处理完成，processing 字段被删除。
![FIFO 存储过程](https://img-blog.csdnimg.cn/20200819213800614.png)
key 为1、2、3的元素入队，当调用 Get 方法，从队列头取出1，放入 processing 集合中表示正在处理该元素，此时又有一个1号元素通过 Add 方法入队，此时 processing 中已经有 1 号元素，所以重复的1号元素不会进入到 queue 中，而是到了 dirty 集合中。等处理完旧的1后，调用 Done 方法标记该元素已经被处理完成，processing 字段被删除。再将新的1放入 queue 中。
![FIFO 并发存储过程](https://img-blog.csdnimg.cn/20200819214407727.png)
## 四、延迟队列
基于 FIFO 队列的接口，加上了 AddAfter 方法，调用该方法，传入时间延迟参数，可以实现延迟入队。

```go
// k8s.io/client-go/util/workqueue/delaying_queue.go
type DelayingInterface interface {
	Interface
	// 指定 duration 时间后将 item 元素插入队列
	AddAfter(item interface{}, duration time.Duration)
}
type delayingType struct {
	Interface
	clock clock.Clock
	stopCh chan struct{}
	heartbeat clock.Ticker

	// 初始大小为1000的缓存 channel，超过1000阻塞
	waitingForAddCh chan *waitFor
	metrics retryMetrics
}
```
![延迟队列运行原理](https://img-blog.csdnimg.cn/20200821212930232.png)
入队时元素先放入 waitingForAddCh 缓存管道中，消费者 waitingLoop 协程不断循环调用 AfterNow 方法从管道中取元素。如果没到延迟时间，则放入左边的优先队列中，如果到时间了则放入 FIFO 队列。同时遍历有序队列，拿出到时间的元素放入 FIFO 队列。
**为什么需要 waitingForAddCh 管道？** 因为 waitingLoop 是协程，协程间通信需要通过管道。
## 五、限速队列
基于延迟队列 DelayingInterface 的接口，加上了 AddRateLimited、Forget、NumRequeues 方法。同时还实现了4种限流算法的接口（RateLimiter），利用延迟队列的特性，延迟元素的入队时间来实现限流。

```go
// k8s.io/client-go/util/workqueue/rate_limitting_queue.go
type RateLimitingInterface interface {
	// 基于延迟队列 DelayingInterface 的接口
	DelayingInterface
	// 入队方法，当 RateLimiter 的限流算法同意新元素入队时，会将该元素入队
	AddRateLimited(item interface{})
	// 指定从 RateLimiter 中释放某元素（清空排队次数）
	Forget(item interface{})
	// 获取某元素的排队次数（相同元素多次入队次数）
	NumRequeues(item interface{}) int
}
// k8s.io/client-go/util/workqueue/default_rate_limiters.go
type RateLimiter interface {
	// 获取指定元素应该等待的时间
	When(item interface{}) time.Duration
	// 指定从 RateLimiter 中释放某元素（清空排队次数）
	Forget(item interface{})
	// 获取某元素的排队次数（相同元素多次入队次数）
	NumRequeues(item interface{}) int
}
```
### 令牌桶算法
![令牌桶算法](https://img-blog.csdnimg.cn/20200821215016510.png)
元素入队前先要从 bucket 桶中获取 token 令牌。bucket 中的令牌放入的速度是有限的，如果元素消耗令牌过快，则会先耗尽 bucket 中的 token。此时需要延迟等待令牌的发放，才能入队，实现了限流。

```go
// k8s.io/client-go/util/workqueue/default_rate_limiters.go
// 初始化令牌桶 RateLimiter
// 表示每秒往 bucket 中放10个 token（10 qps），bucket 大小为100
BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)}

func (r *BucketRateLimiter) When(item interface{}) time.Duration {
	return r.Limiter.Reserve().Delay()
}

// golang.org\x\time\rate\rate.go
// 算延迟后的时间
func (lim *Limiter) Reserve() *Reservation {
	return lim.ReserveN(time.Now(), 1)
}
// 算出延迟时间距离当前时间的时间差
func (r *Reservation) Delay() time.Duration {
	return r.DelayFrom(time.Now())
}
func (r *Reservation) DelayFrom(now time.Time) time.Duration {
	if !r.ok {
		return InfDuration
	}
	// 算时间差
	delay := r.timeToAct.Sub(now)
	if delay < 0 {
		return 0
	}
	return delay
}
```

### 排队指数算法
排队指数指相同元素的排队次数，每次调用 AddRateLimited 函数入队则排一次队。相同元素多次插入队列的延迟时间会呈指数增长，限制了相同元素的入队次数。
```go
// k8s.io/client-go/util/workqueue/default_rate_limiters.go
func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	...
	// 每次调 AddRateLimited 该元素的排队次数加一
	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1

	// 每次将排队时延按指数增长，最长不超过 maxDelay
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay {
		return r.maxDelay
	}

	return calculated
}
```

### 计数器算法
限制一段时间内允许通过的元素数量。每次元素入队，计数器加一，计数器达到阈值则减慢元素入队速度（入队时延加大）。等待限速时间窗口过去之后计数器清零，新的元素继续入队。

```go
// k8s.io/client-go/util/workqueue/default_rate_limiters.go
func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
	...
	// 相同元素的入队计数器
	r.failures[item] = r.failures[item] + 1
	// 小于阈值，则快速入队（等待时间小）
	if r.failures[item] <= r.maxFastAttempts {
		return r.fastDelay
	}
	// 大于阈值，则减速入队（等待时间长）
	return r.slowDelay
}
```
### 混合模式
多种限流算法混合使用

```go
// 初始化令牌桶算法和排队指数算法
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		// 表示排队指数算法的入队时延范围为 5ms 到 1000s
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 表示每秒往 bucket 中放10个 token（10 qps），bucket 大小为100
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}
```
