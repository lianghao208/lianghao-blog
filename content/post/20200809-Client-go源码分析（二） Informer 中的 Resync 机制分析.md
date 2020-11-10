---
title: "Client-go源码分析（二） Informer 中的 Resync 机制分析"
date: 2020-08-09T14:24:26+08:00
draft: false
---
## 一、Client-go 中的 Informer 工作流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200809094552541.png)
Informer 中的 Reflector 通过 List/watch 从 apiserver 中获取到集群中所有资源对象的变化事件（event），将其放入 Delta FIFO 队列中（以 Key、Value 的形式保存），触发 onAdd、onUpdate、onDelete 回调将 Key 放入 WorkQueue 中。同时将 Key 更新 Indexer 本地缓存。Control Loop 从 WorkQueue 中取到 Key，从 Indexer 中获取到该 Key 的 Value，进行相应的处理。
## 二、Resync 机制的引入
我们在使用 SharedInformerFactory 去创建 SharedInformer 时，需要填一个 ResyncDuration 的参数

```go
// k8s.io/client-go/informers/factory.go
// NewSharedInformerFactory constructs a new instance of sharedInformerFactory for all namespaces.
func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration) SharedInformerFactory {
	return NewSharedInformerFactoryWithOptions(client, defaultResync)
}
```
这个参数指的是，多久从 Indexer 缓存中同步一次数据到 Delta FIFO 队列，重新走一遍流程

```go
// k8s.io/client-go/tools/cache/delta_fifo.go
// 重新同步一次 Indexer 缓存数据到 Delta FIFO 队列中
func (f *DeltaFIFO) Resync() error {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.knownObjects == nil {
		return nil
	}
	// 遍历 indexer 中的 key，传入 syncKeyLocked 中处理
	keys := f.knownObjects.ListKeys()
	for _, k := range keys {
		if err := f.syncKeyLocked(k); err != nil {
			return err
		}
	}
	return nil
}

func (f *DeltaFIFO) syncKeyLocked(key string) error {
	obj, exists, err := f.knownObjects.GetByKey(key)
	if err != nil {
		klog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
		return nil
	} else if !exists {
		klog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
		return nil
	}
	// 如果发现 FIFO 队列中已经有相同 key 的 event 进来了，说明该资源对象有了新的 event，
	// 在 Indexer 中旧的缓存应该失效，因此不做 Resync 处理直接返回 nil
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	if len(f.items[id]) > 0 {
		return nil
	}
    // 重新放入 FIFO 队列中
	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}

```
为什么需要 Resync 机制呢？因为在处理 SharedInformer 事件回调时，可能存在处理失败的情况，定时的 Resync 让这些处理失败的事件有了重新处理的机会。
那么经过 Resync 重新放入 Delta FIFO 队列的事件，和直接从 apiserver 中 watch 得到的事件处理起来有什么不一样呢？

```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		// 判断事件类型，看事件是通过新增、更新、替换、删除还是 Resync 重新同步产生的
		switch d.Type {
		case Sync, Replaced, Added, Updated:
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				
				isSync := false
				switch {
				case d.Type == Sync:
					// 如果是通过 Resync 重新同步得到的事件则做个标记
					isSync = true
				case d.Type == Replaced:
					...
				}
				// 如果是通过 Resync 重新同步得到的事件，则触发 onUpdate 回调
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, false)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```
从上面对 Delta FIFO 的队列处理源码可看出，如果是从 Resync 重新同步到 Delta FIFO 队列的事件，会分发到 updateNotification 中触发 onUpdate 的回调
## 三、总结
Resync 机制的引入，定时将 Indexer 缓存事件重新同步到 Delta FIFO 队列中，在处理 SharedInformer 事件回调时，让处理失败的事件得到重新处理。并且通过入队前判断 FIFO 队列中是否已经有了更新版本的 event，来决定是否丢弃 Indexer 缓存不进行 Resync 入队。在处理 Delta FIFO 队列中的 Resync 的事件数据时，触发 onUpdate 回调来让事件重新处理。