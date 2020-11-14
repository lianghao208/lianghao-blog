---
title: "K8S调度器Schduler工作原理--Predicates预选算法过程分析"
date: 2020-01-14T14:24:26+08:00
draft: false
tags:
    - Kubernetes
    - Scheduler
---

# Scheduler工作流程
我们在使用K8S集群时，常常需要对Deployment Controller做创建、修改、删除操作，K8S对应的会创建、销毁和重新调度Pod在合适的节点上，这个调度过程是通过K8S的Scheduler调度器实现的。Schduler的工作流程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200111162951813.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
Informer组件一直在监听etcd中Pod信息的变化，准确来说，监听的是Pod信息中Spec.nodeName字段的变化，一旦检测到该字段为空，则认为集群中有Pod尚未调度到Node中，这时Informer开始将这个Pod的信息加入队列中，同时更新Scheduler Cache缓存。接下来Pod信息从队列中出队，进入Predicates（预选阶段），该阶段通过一系列的预选算法选出集群中适合Pod运行的节点，带着这些信息进入Priorities（优选阶段）。同理，该阶段通过一系列的优选算法为适合该Pod调度对每个Node进行打分，最后选出集群中最适合（也就是分数最高的）Pod运行的**一个节点**，最后将这个节点和Pod进行绑定（Bind），更新缓存，从而实现Pod的调度。

# Predicates预选流程
predicates 算法主要是对集群中的 node 进行过滤，选出符合当前 pod 运行的一组 nodes。过滤 node 的预选算法有很多，比如：CheckNodeConditionPredicate（检查节点是否可被调度），PodFitsHost（检查pod.spec.nodeName字段是否已经指定），PodFitsHostPorts（检查pod需要的端口node能否提供）等，预选算法执行顺序如下：

```go
var (
   predicatesOrdering = []string{
       CheckNodeConditionPred, 
       CheckNodeUnschedulablePred,
       GeneralPred, 
       HostNamePred, 
       PodFitsHostPortsPred,
       MatchNodeSelectorPred, 
       PodFitsResourcesPred, 
       NoDiskConflictPred,
       PodToleratesNodeTaintsPred, 
       PodToleratesNodeNoExecuteTaintsPred, 
       CheckNodeLabelPresencePred,
       CheckServiceAffinityPred, 
       MaxEBSVolumeCountPred, 
       MaxGCEPDVolumeCountPred, 
       MaxCSIVolumeCountPred,
       MaxAzureDiskVolumeCountPred, 
       CheckVolumeBindingPred, 
       NoVolumeZoneConflictPred,
       CheckNodeMemoryPressurePred, 
       CheckNodePIDPressurePred, 
       CheckNodeDiskPressurePred, 
       MatchInterPodAffinityPred}
)
```
这个顺序是可以被配置文件覆盖的，用户可以指定类似于这样的配置文件修改预选算法执行顺序：

```yaml
{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
    {"name" : "PodFitsHostPorts", "order": 2},
    {"name" : "PodFitsResources", "order": 3},
    {"name" : "NoDiskConflict", "order": 5},
    {"name" : "PodToleratesNodeTaints", "order": 4},
    {"name" : "MatchNodeSelector", "order": 6},
    {"name" : "PodFitsHost", "order": 1}
    ],
"priorities" : [
    {"name" : "LeastRequestedPriority", "weight" : 1},
    {"name" : "BalancedResourceAllocation", "weight" : 1},
    {"name" : "ServiceSpreadingPriority", "weight" : 1},
    {"name" : "EqualPriority", "weight" : 1}
    ],
"hardPodAffinitySymmetricWeight" : 10
}
```

对于一个 pod，一组 node 并发执行 predicates 预选算法（注意，这里是多个 node 同时**顺序执行**这一组算法，而不是一个node同时执行多个算法），一旦某个 node 顺序执行预选算法的过程中，某个算法执行失败了，这个 node 直接被踢出，不再执行后面的流程。
经过这样的过滤，实现 node 的预选过程，最终剩下的一组 node 将会进入下一阶段：priorities 优选过程。关于优选过程再下一章节会详细介绍。


# predicate的并发过程
预选算法的执行过程是通过多个 node 并发执行实现的：

***pkg/scheduler/core/generic_scheduler.go:389***
```go
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
    checkNode := func(i int) {
    	//podFitsOnNode里就是执行预选算法的逻辑
        fits, failedPredicates, err := podFitsOnNode(
            //……
        )
        if fits {
            length := atomic.AddInt32(&filteredLen, 1)
            filtered[length-1] = g.cachedNodeInfoMap[nodeName].Node()
        } 
    }
    workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode)

    if len(filtered) > 0 && len(g.extenders) != 0 {
        for _, extender := range g.extenders {
            // Logic of extenders 
        }
    }
    return filtered, failedPredicateMap, nil
}
```
workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode) 函数用于是并发执行N个独立的工作过程的，这里指的是开启16个go协程并发处理 int(allNodes) 个（也就是node的数量）任务，每个任务的内容就是调用 checkNode 方法。从上述代码中可见，调用 checkNode 方法就是调用了里面的 podFitsOnNode 方法，也就执行了预选算法的逻辑。
下面详细讲解 podFitsOnNode 方法的实现。

# 用 podFitsOnNode 函数实现一个node的预选过程
***pkg/scheduler/core/generic_scheduler.go:425***
```go
func podFitsOnNode(
    pod *v1.Pod,
    meta algorithm.PredicateMetadata,
    info *schedulercache.NodeInfo,
    predicateFuncs map[string]algorithm.FitPredicate,
    nodeCache *equivalence.NodeCache,
    queue internalqueue.SchedulingQueue,
    alwaysCheckAllPredicates bool,
    equivClass *equivalence.Class,
) (bool, []algorithm.PredicateFailureReason, error) {
    podsAdded := false
    //这里循环调用了两次预选算法，原因涉及到了pod的抢占逻辑，后面会详细说明
    for i := 0; i < 2; i++ {
        metaToUse := meta
        nodeInfoToUse := info
        //只有第一次循环才会调addNominatedPods函数
        if i == 0 {
            podsAdded, metaToUse, nodeInfoToUse = addNominatedPods(pod, meta, info, queue)
        } else if !podsAdded || len(failedPredicates) != 0 {
            break
        }
        eCacheAvailable = equivClass != nil && nodeCache != nil && !podsAdded
		// predicates.Ordering()得到的是一个[]string，predicate名字集合
		for predicateID, predicateKey := range predicates.Ordering() {
		    var (
		        fit     bool
		        reasons []algorithm.PredicateFailureReason
		        err     error
		    )
		    // 如果predicateFuncs有这个key，则调用这个predicate；也就是说predicateFuncs如果随便定义了其他名字，会被忽略，因为predicateKey是内部指定的。
		    if predicate, exist := predicateFuncs[predicateKey]; exist {
		        if eCacheAvailable {
		            fit, reasons, err = nodeCache.RunPredicate(predicate, predicateKey, predicateID, pod, metaToUse, nodeInfoToUse, equivClass)
		        } else {
		            // 这里真正调用predicate函数
		            fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)
		        }
		        if err != nil {
		            return false, []algorithm.PredicateFailureReason{}, err
		        }
		        if !fit {
		            // ……
		        }
		    }
		}
    }

    return len(failedPredicates) == 0, failedPredicates, nil
}
```
以上代码可以分为两部分：
**一、在第一次for循环里面调用了addNominatedPods函数，这个函数将更高优先级的抢占类型的Pod先加入了Scheduler缓存。
二、第一次和第二次循环都调用了predicate函数，这里真正执行了Predicates预选算法的逻辑。**
那么为什么要循环两次呢？因为第一次循环和第二次循环所做的操作略有不同。从代码中可以看出，第一次循环比第二次多调了addNominatedPods函数。这个函数将pod和当前在等待调度的pod队列传入，筛选出队列中满足pod.Status.NominatedNodeName字段不为空且优先级大于等于当前pod的pod信息，加入Scheduler缓存。
也就是下面两个条件pod要同时满足，才会把pod加入Scheduler缓存：
**一、pod.Status.NominatedNodeName字段不为空
二、优先级大于等于当前pod**
将这种类型的pod加入Scheduler缓存再进行预选算法的原因是：
在pod抢占node的逻辑中，优先级高的pod先抢占node，抢占成功后将pod.Status.NominatedNodeName字段设置成当前的node，设置完成后scheduler就跑去执行下一个pod的调度逻辑了，这时pod很可能还没有真正在node上面跑起来（不一定是running状态）。所以Scheduler缓存中其实并没有将这类pod的信息，所以在调度当前pod的时候，会受这些高优先级pod的影响（pod和pod之间有pod亲和性、反亲和性等依赖关系），**所以要先假设这类高优先级的pod已经在这个node中跑起来了**。也就有了手动调用addNominatedPods函数把其他高优先级的pod加入了缓存再执行预选算法这两个过程。
不仅如此，为了确保万无一失（万一这些高优先级的pod最终没在这个node跑起来），还得把这些高优先的pod排除掉再执行一次预选算法。
这样，无论其它高优先级的pod在不在这个node上，这个pod都能确保无冲突地调度在这些node上面。

# 单个predicate执行过程
predicate函数是一系列预选算法的集合：

```go
```go
var (
   predicatesOrdering = []string{
       CheckNodeConditionPred, 
       CheckNodeUnschedulablePred,
       GeneralPred, 
       HostNamePred, 
       PodFitsHostPortsPred,
       MatchNodeSelectorPred, 
       PodFitsResourcesPred, 
       NoDiskConflictPred,
       PodToleratesNodeTaintsPred, 
       PodToleratesNodeNoExecuteTaintsPred, 
       CheckNodeLabelPresencePred,
       CheckServiceAffinityPred, 
       MaxEBSVolumeCountPred, 
       MaxGCEPDVolumeCountPred, 
       MaxCSIVolumeCountPred,
       MaxAzureDiskVolumeCountPred, 
       CheckVolumeBindingPred, 
       NoVolumeZoneConflictPred,
       CheckNodeMemoryPressurePred, 
       CheckNodePIDPressurePred, 
       CheckNodeDiskPressurePred, 
       MatchInterPodAffinityPred}
)
```
predicatesOrdering 在轮询这个数组中的函数名进行顺序调用，下面举例说明几个预选算法：

一、NoDiskConflict：判断当前pod的卷信息是否和node的卷信息有冲突
***pkg/scheduler/algorithm/predicates/predicates.go:277***
```go
func NoDiskConflict(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
    for _, v := range pod.Spec.Volumes {
        for _, ev := range nodeInfo.Pods() {
            if isVolumeConflict(v, ev) {
                return false, []algorithm.PredicateFailureReason{ErrDiskConflict}, nil
            }
        }
    }
    return true, nil, nil
}
```
二、PodMatchNodeSelector：判断当前pod的nodeSelector字段和node信息是否匹配
***pkg/scheduler/algorithm/predicates/predicates.go:852***
```go
// PodMatchNodeSelector checks if a pod node selector matches the node label.
func PodMatchNodeSelector(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
	node := nodeInfo.Node()
	if node == nil {
		return false, nil, fmt.Errorf("node not found")
	}
	if podMatchesNodeSelectorAndAffinityTerms(pod, node) {
		return true, nil, nil
	}
	return false, []algorithm.PredicateFailureReason{ErrNodeSelectorNotMatch}, nil
}
```
三、PodFitsHost：判断当前pod的Spec.NodeName字段和node的名字是否匹配
***pkg/scheduler/algorithm/predicates/predicates.go:864***
```go
// PodFitsHost checks if a pod spec node name matches the current node.
func PodFitsHost(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
	if len(pod.Spec.NodeName) == 0 {
		return true, nil, nil
	}
	node := nodeInfo.Node()
	if node == nil {
		return false, nil, fmt.Errorf("node not found")
	}
	if pod.Spec.NodeName == node.Name {
		return true, nil, nil
	}
	return false, []algorithm.PredicateFailureReason{ErrPodNotMatchHostName}, nil
}
```
到这里，Predicates 预选算法执行的逻辑就结束了，后面会进入 Priorities 优选算法的执行逻辑。