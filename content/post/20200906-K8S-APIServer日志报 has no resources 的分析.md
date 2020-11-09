---
title: "K8S-APIServer日志报 has no resources 的分析"
date: 2020-09-06T14:24:26+08:00
draft: false
---

## 一、现象
在生产环境中遇到 api-server 服务发生陆续重启的现象， 查看监控，APIServer 所在的 master 节点的CPU、内存和网络流量发生抖动。
在 APIServer 日志中可以看到日志中存在 it has no resources 的警告日志：

```bash
{"log":"W0814 03:04:44.058851       1 genericapiserver.go:342] Skipping API image.openshift.io/1.0 because it has no resources.\n","stream":"stderr","time":"2020-08-14T03:04:44.059033849Z"}

{"log":"W0814 03:04:44.058851       1 genericapiserver.go:342] Skipping API image.openshift.io/1.0 because it has no resources.\n","stream":"stderr","time":"2020-08-14T03:04:44.059033849Z"}

{"log":"W0814 03:04:49.422600       1 genericapiserver.go:342] Skipping API pre012 because it has no resources.\n","stream":"stderr","time":"2020-08-14T03:04:49.422708175Z"}
```
## 二、源码分析
查阅源码，警告日志的地方发生在 staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go 文件中的 installAPIResources 方法：

```go
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
		if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
			glog.Warningf("Skipping API %v because it has no resources.", groupVersion)
			continue
		}
		...
	}
	return nil
}
```
apiGroupInfo 的 VersionedResourcesStorageMap 字段用于存储**资源版本**、**资源**和**资源存储对象**的映射关系，其表现形式为 map[string]map[string]rest.Storage，比如 Pod 资源与资源对象的映射关系为：v1/pods/PodStorage （资源版本/资源/资源存储对象）。
那么资源存储对象是什么呢？

```go
type PodStorage struct {
	Pod         *REST
	Binding     *BindingREST
	Eviction    *EvictionREST
	Status      *StatusREST
	Log         *podrest.LogREST
	Proxy       *podrest.ProxyREST
	Exec        *podrest.ExecREST
	Attach      *podrest.AttachREST
	PortForward *podrest.PortForwardREST
}
```
以 PodStorage 为例，资源存储对象其实是一个描述资源对象的数据结构，用于保存从 etcd 中查出的资源对象数据。
这段代码的逻辑为：遍历 PrioritizedVersions（Kubernetes 所支持的所有资源版本列表） ，取出每个 groupVersion（版本列表），在 VersionedResourcesStorageMap 中找对应的**资源存储对象**，如果没有找到（没有启用该资源版本），则报 **Skipping API  ... because it has no resources.** 。

 1. 列出 Kubernetes 支持的所有资源 ==》PrioritizedVersions 
 2. 实例化 APIGroupInfo ==》VersionedResourcesStorageMap
 3. GenericAPIServer 中创建资源存储对象的 API 映射：遍历 PrioritizedVersions，判断 VersionedResourcesStorageMap 中是否有 PrioritizedVersions 资源列表中对应的资源存储对象（如果没有创建则报 **it has no resources** ）

因此，生产上的警告日志的意思是：集群中有 image.openshift.io/1.0 这个资源的Group版本，但是**没有找到对应的资源存储对象**（由于没有启用该资源版本导致，后面会详细说明），所以跳过为这个资源存储对象创建 API 的操作。

这个告警仅仅用于提示集群中没有启用某个资源版本，属于正常现象。
## 三、APIServer的启动流程
上面提到的警告日志发生在 APIServer 的启动时期，因此这里整理了 APIServer 的启动流程

 1. 资源注册
 2. 创建 APIServer 通用配置
 3. 创建 APIExtentionsServer
 4. 创建 KubeAPIServer
 5. 创建 AggregatorServer
 6. 创建 GenericAPIServer
 7. 启动 HTTP 服务
 8. 启动 HTTPS 服务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200816111138617.png)

APIServer 本质上就是一个提供 RESTful API 的 web 服务，在启动 HTTP/HTTPS 服务之前做的一系列工作都是为 Kubernetes 集群中的资源对象创建路由和 Etcd 的绑定关系。
例如：我们要通过 APIServer 访问某个 namespace 下 Pod 的信息(内部资源对象)，就需要访问 APIServer 的 /api/v1/namespaces/{namespace}/pods 接口，而这个接口的路由映射就是通过 APIServer 启动时在创建 KubeAPIServer 这一步生成的。

### 对应的源码（以 batch 资源为例）：
#### 1、资源注册
在资源注册时，Kubernetes 将所支持的所有资源版本类型放入 Scheme 对象中
batch 支持 v1、v1beta1 和 v2alpha1 三个版本
```go
// pkg/apis/batch/install/install.go
func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(batch.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v2alpha1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta1.SchemeGroupVersion, v2alpha1.SchemeGroupVersion))
}
```
#### 2、创建 KubeAPIServer
```go
// pkg/master/master.go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {
	...
	// 1、创建 GenericAPIServer
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	...
	// 2、初始化 Mater
	m := &Master{
		GenericAPIServer: s,
	}
	...
	// 3、InstallLegacyAPI 注册 /api 资源
	m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
	}
	...
	// 4、InstallAPIs 注册 /apis 资源
	m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)
	...
	return m, nil
}
```
第2步中初始化 Mater，里面默认启用了一些资源版本，也默认禁用了一些资源版本。而禁用的版本列表就是日志报 **it has no resources** 的关键！！

```go
// pkg/master/master.go
func DefaultAPIResourceConfigSource() *serverstorage.ResourceConfig {
	ret := serverstorage.NewResourceConfig()
	// 这里默认启用了一些资源版本
	ret.EnableVersions(
		admissionregistrationv1beta1.SchemeGroupVersion,
		apiv1.SchemeGroupVersion,
		appsv1beta1.SchemeGroupVersion,
		appsv1beta2.SchemeGroupVersion,
		appsv1.SchemeGroupVersion,
		authenticationv1.SchemeGroupVersion,
		...
	)
	// 这里默认禁用了一些资源版本
	ret.DisableVersions(
		admissionregistrationv1alpha1.SchemeGroupVersion,
		batchapiv2alpha1.SchemeGroupVersion,
		rbacv1alpha1.SchemeGroupVersion,
		schedulingv1alpha1.SchemeGroupVersion,
		settingsv1alpha1.SchemeGroupVersion,
		storageapiv1alpha1.SchemeGroupVersion,
	)

	return ret
}
```

第4步中调用了 InstallAPIs 方法，方法中做了两件事
1、实例化 APIGroupInfo 对象，也就是为我们前面提到的 VersionedResourcesStorageMap 建立**资源版本**、**资源**和**资源存储对象**的映射关系。由于默认禁用了 batch 的 v2alpha1 版本，所以不会建立它的映射关系，**最终 apiserver 启动时会报找不到 batch/v2alpha1 的 resources**

```go
// pkg/registry/batch/rest/storage_batch.go
func (p RESTStorageProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool) {
	// 实例化 APIGroupInfo 对象
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(batch.GroupName, legacyscheme.Scheme, legacyscheme.ParameterCodec, legacyscheme.Codecs)
	// 这里对资源的各个版本是否启用做了判断
	if apiResourceConfigSource.VersionEnabled(batchapiv1.SchemeGroupVersion) {
		apiGroupInfo.VersionedResourcesStorageMap[batchapiv1.SchemeGroupVersion.Version] = p.v1Storage(apiResourceConfigSource, restOptionsGetter)
	}
	if apiResourceConfigSource.VersionEnabled(batchapiv1beta1.SchemeGroupVersion) {
		apiGroupInfo.VersionedResourcesStorageMap[batchapiv1beta1.SchemeGroupVersion.Version] = p.v1beta1Storage(apiResourceConfigSource, restOptionsGetter)
	}
	// 在第2步初始化 Mater 时，禁用了 batch 的 v2alpha1 版本，这里不会将 batchapiv2alpha1 的资源存储对象放入 Map 中，因此 apiserver 启动会报找不到资源的警告
	if apiResourceConfigSource.VersionEnabled(batchapiv2alpha1.SchemeGroupVersion) {
		apiGroupInfo.VersionedResourcesStorageMap[batchapiv2alpha1.SchemeGroupVersion.Version] = p.v2alpha1Storage(apiResourceConfigSource, restOptionsGetter)
	}

	return apiGroupInfo, true
}
```
2、通过 InstallAPIGroup 方法将 APIGroupInfo 对象中的 VersionedResourcesStorageMap （也就是**资源版本**、**资源**和**资源存储对象**的映射关系）注册到 KubeAPIServer Handlers 方法中。

```go
func (s *GenericAPIServer) InstallAPIGroup(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
	...
	// 这里为 Kubernetes 的资源对象创建 资源版本/资源/资源存储对象 到 HTTP 的请求路径的映射关系
	if err := s.installAPIResources(APIGroupPrefix, apiGroupInfo); err != nil {
		return err
	}
	...
	return nil
```
上面的 installAPIResources 方法执行逻辑就是开头提到的逻辑：

> 遍历 PrioritizedVersions（Kubernetes 所支持的所有资源版本列表） ，取出每个
> groupVersion（版本列表），在 VersionedResourcesStorageMap
> 中找对应的**资源存储对象**，如果没有找到，则报 **Skipping API  ... because it has no
> resources.** 

查看 apisever 的启动日志，发现果然打印出了 **batch/v2alpha1 has no resources** 的日志
```bash
W0816 05:14:24.570339       1 genericapiserver.go:319] Skipping API batch/v2alpha1 because it has no resources.
W0816 05:14:25.277714       1 genericapiserver.go:319] Skipping API rbac.authorization.k8s.io/v1alpha1 because it has no resources.
W0816 05:14:25.288664       1 genericapiserver.go:319] Skipping API scheduling.k8s.io/v1alpha1 because it has no resources.
W0816 05:14:25.377954       1 genericapiserver.go:319] Skipping API storage.k8s.io/v1alpha1 because it has no resources.
W0816 05:14:26.780652       1 genericapiserver.go:319] Skipping API admissionregistration.k8s.io/v1alpha1 because it has no 
```
那么为什么要默认禁用某些资源版本呢？而且默认禁用的都是 alpha 版本的资源。
Kubernetes 的资源版本分为3种：Alpha、Beta、Stable，它们之间的迭代顺序为Alpha-》Beta-》Stable。其中 Alpha 为第一阶段的版本，一般用于内部测试。Beta 为第二阶段版本，修复了大部分的不完善之处，但仍可能存在bug。Stable 为稳定运行的版本。
因此，Alpha 版本做为内部测试版，存在较多的不稳定因素，官方随时可能放弃支持该版本，因此默认情况下会禁用。
