---
title: "Client-go源码分析（三）Informer机制"
date: 2020-08-12T14:24:26+08:00
draft: false
tags:
    - Kubernetes
    - Informer
---

## 一、简介
Kubernetes 中，控制器需要对集群中的资源对象的状态进行监控，使资源对象的**实际状态**和通过 yaml 定义的**期望状态**协调达到一致。那么控制器是如何对资源对象进行监控，并根据对象的实际状态变化做出相应的处理呢？实际上就是通过 Client-go 包中的 Informer 机制实现的。
![Informer 工作流程架构图](https://img-blog.csdnimg.cn/20200810221241261.png)
图片来源：极客时间 --《深入浅出 Kubernetes》
从上图中，我们可以大致了解到 Informer 机制的整个流程：

- Reflector 组件通过对 apiserver 发起 HTTP 请求，List 得到集群中所有资源对象，再通过 HTTP 长连接 Watch 不断监听得到 etcd 变化的数据，封装成一个个事件（Event），再将事件对应的资源对象放入 Delta FIFO 队列中
- Delta FIFO 队列保存了需要操作的资源对象，以及要对该资源对象进行什么操作（Add、Updated、Deleted、Sync）
- Indexer 是一个带索引的本地缓存，它从Delta FIFO 队列取出资源对象的索引（和 etcd 的 key 保持一致）和具体数据进行保存
- 从Delta FIFO 队列中取出的资源对象还会触发三种方法的回调：Add、Delete、Update 分别对应 etcd 中资源对象的增、删、改操作。回调方法中会将需要处理的资源对象的索引（Index）放入 WorkQueue 工作队列中
- Control Loop 控制循环不断地从 WorkQueue 中取出要处理对象的索引，通过 Lister 从 Indexer 维护的本地缓存中获取具体的资源对象数据，进行相应的处理。

## 二、Reflector 底层原理分析
简单来说，Reflector 就做了两件事：List 和 Watch

```go
// k8s.io/client-go/tools/cache/reflector.go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	...
	if err := func() error {
		...
		pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
				return r.listerWatcher.List(opts)
			}))
		...
		w, err := r.listerWatcher.Watch(options)
		...
}
```
List 指的是向 apiserver 发送 GET 请求获取集群中所有资源的数据，Watch 则是通过与 apiserver 建立 HTTP 长连接并通过分块编码传输（chunked）来监控资源对象的变更事件。而这些请求的发送都是通过我们前面提到的 ClientSet 客户端实现的。
[分块编码传输简介](https://foofish.net/http-transfer-encoding.html)

```go
// 提供了 List 和 Watch 接口，让资源对象去实现具体的接口
type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// 是否禁用 watch 时的分块编码传输
	DisableChunking bool
}

type ListFunc func(options metav1.ListOptions) (runtime.Object, error)

type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)

// 以 Pod 资源对象为例，底层通过 ClientSet 客户端向 apiserver 发送请求
// k8s.io/client-go/informers/core/v1/pod.go
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```
当 Watch 监控到集群中有资源对象变化的事件时，会触发 watchHandler 回调，将变化的资源对象放入 Delta  FIFO 中，同时更新本地缓存。

```go
// k8s.io/client-go/tools/cache/reflector.go
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
		...
		select {
		...
		// 有资源对象变化事件时，channel 被触发
		case event, ok := <-w.ResultChan():
			...
			// 这里更新本地缓存
			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
				...
			case watch.Modified:
				err := r.store.Update(event.Object)
				...
			case watch.Deleted:
				err := r.store.Delete(event.Object)
				...
			}
			*resourceVersion = newResourceVersion
			// 这里更新资源版本号
			r.setLastSyncResourceVersion(newResourceVersion)
		}
	}
	return nil
}
```
## 三、Delta FIFO 底层原理分析 
Delta FIFO 数据结构：

```go
// k8s.io/client-go/tools/cache/delta_fifo.go
type DeltaFIFO struct {
	...
	items map[string]Deltas
	queue []string
	...
}
```

可以看到其主要由两部分组成：FIFO 队列（切片），Deltas 对象集合。
Delta 对象保存了资源对象和对象的操作类型（Add、Update、Delete、Sync）
而 FIFO 队列则负责存放数据生产者 Reflector 中取到的数据，然后消费者 Controller 再从队列头部取出数据。
入队：

```go
// k8s.io/client-go/tools/cache/delta_fifo.go
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	// 为资源对象算出 key，作为 indexer 本地缓存对象的索引
	id, err := f.KeyOf(obj)
	...
	// 将资源对象 obj 和对象的操作类型 actionType 封装成完整的 Delta 对象
	// 加入 FIFO 队列中
	newDeltas := append(f.items[id], Delta{actionType, obj})
	// 这里对最后入队的连续两个相同操作的 Delta 对象进行去重
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		f.items[id] = newDeltas
		// 广播通知所有消费者解除阻塞状态（用 runtime 包的 runtime_notifyListNotifyAll 实现唤醒）
		f.cond.Broadcast()
	}
	...
	return nil
}
```
出队：

```go
// k8s.io/client-go/tools/cache/delta_fifo.go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	// 加锁，保证队列的并发操作线程安全
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			...
			// FIFO 队列为空，则阻塞等待
			f.cond.Wait()
		}
		// 头部的对象出队
		id := f.queue[0]
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		// 判断 Delta FIFO 队列中该对象是否已经被删除，删除则忽略
		item, ok := f.items[id]
		if !ok {
			continue
		}
		delete(f.items, id)
		// Delta FIFO 队列中对象仍然存在，则调用 process 函数触发对象的回调方法
		err := process(item)
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		return item, err
	}
}
```
Resync 机制：（定时从 Indexer 的本地缓存中将数据再次放入 FIFO 队列的同步机制）
Resync 由 Reflector 中的管道触发，因为在创建 Reflector 时，NewRelector函数传入了 resyncPeriod 参数，设定了 Resync 的触发时间周期。
```go
// k8s.io/client-go/tools/cache/reflector.go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    ...
    go func() {
            resyncCh, cleanup := r.resyncChan()
            defer func() {
                cleanup()
            }()
            for {
                select {
                // Resync 管道触发
                case <-resyncCh:
                case <-stopCh:
                    return
                case <-cancelCh:
                    return
                }
                if r.ShouldResync == nil || r.ShouldResync() {
                    klog.V(4).Infof("%s: forcing resync", r.name)
                    //执行Resync操作
                    if err := r.store.Resync(); err != nil {
                        resyncerrc <- err
                        return
                    }
                }
                cleanup()
                resyncCh, cleanup = r.resyncChan()
            }
        }()
    ......
}
```
Resyn 中处理 Indexer 缓存和 Delta FIFO 队列同步的逻辑：
```go
// k8s.io/client-go/tools/cache/delta_fifo.go
func (f *DeltaFIFO) syncKeyLocked(key string) error {
	// 从 Indexer 的本地存储中根据索引取对象
	obj, exists, err := f.knownObjects.GetByKey(key)
	...

	// 为了防止 Resync 时将旧版本的对象覆盖掉了还在 Delta FIFO 队列中新版本的对象
	// 因此需要先看看 Delta FIFO 队列中有没有新对象和它竞争，有的话直接忽略不做这个
	// 对象的 Resync 同步
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	if len(f.items[id]) > 0 {
		return nil
	}
	// 再次进入 FIFO 队列，但 actionType 标记成了 Sync
	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}

```
## 四、Indexer 底层实现
Indexer 是用于存 Delta FIFO 队列中消费出来的 Kubernetes 资源对象的本地缓存，它通过索引将每个出队的资源对象保存起来，方便 Controller 回调时获取。既然是一种通过索引进行查找的数据结构，我们很容易想到 Map 这一种数据结构。没错，Indexer 的底层就是通过一个叫 ThreadSafeMap的结构体实现的，该结构体中有一个 map 字段，用来存资源对象的数据。

```go
// k8s.io/client-go/tools/cache/thread_safe_store_test.go
type threadSafeMap struct {
	...
	items map[string]interface{}
	// 存储的索引器，key 为索引器名字，value 为索引器的实现
	indexers Indexers
	// 存储的缓存器，key 为缓存器名字，value 为缓存的对象数据
	indices Indices
}

// k8s.io/client-go/tools/cache/index.go
// 存储缓存数据的数据结构（K/V）
type Index map[string]sets.String

// 存储的索引器，key 为索引器名字，value 为索引器的实现
type Indexers map[string]IndexFunc

// 存储的缓存器，key 为缓存器名字，value 为缓存的对象数据
type Indices map[string]Index
```
如何通过 Indexer 拿到索引数据呢？threadSafeMap 有 ByIndex 方法，传入索引器名称（indexName）和数据的索引（indexKey）即可从缓存中筛选出数据：

```go
// k8s.io/client-go/tools/cache/thread_safe_store_test.go
func (c *threadSafeMap) ByIndex(indexName, indexedValue string) ([]interface{}, error) {
	...
	// 通过索引器名称（indexName）从 indexers 中拿到索引器的具体实现
	indexFunc := c.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}
	// 通过缓存器名称（indexName）从 indices 中拿到缓存的对象数据
	index := c.indices[indexName]
	// 再通过数据的索引（indexedValue）从 index 中拿到具体的缓存对象数据
	set := index[indexedValue]
	list := make([]interface{}, 0, set.Len())
	for key := range set {
		list = append(list, c.items[key])
	}

	return list, nil
}
```

## [思考] Informer 中的优化机制
1、Shared Informer 共享机制
同类型的资源对象共享一个 Informer，因此同类型的资源对象获取后只需要进行一次 apiserver 的 List 和 Watch 请求，大大减轻了 apiserver 和 etcd 的负担。
2、Indexer 缓存机制
Indexer 中维护了 Delta FIFO 队列的一份本地缓存，Control Loop 通过索引可直接从缓存中获取资源对象的数据，不需要再和 apiserver 进行交互，减轻的 apiserver 和 etcd 的负担。
3、Resync 机制
为了防止在处理 Informer 事件回调时，处理失败， Delta FIFO 中数据被取走了，却没有正常被执行的情况出现。resync 机制会定时将 Indexer 的缓存数据重新同步到 Delta FIFO 队列中，再次触发 update 回调。（边缘驱动触发+水平驱动触发+定时同步）
4、在 WorkQueue 中只将索引入队，而不将整个资源对象放入工作队列中。一方面节省了内存空间的消耗，另一方面，后期 Control Loop 通过索引访问本地缓存时，可以对资源对象的数据做实时校验（如发现索引在 Indexer 缓存中找不到对应的对象，则说明该资源对象当前已被删除，Control Loop 可以实时获取到对象最新的状态），不会因为 Control Loop 执行时间滞后而导致状态变化有较大延迟
