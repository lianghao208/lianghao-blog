---
title: "K8S-APIServer源码分析组件构成、原理、程序流程"
date: 2020-05-10T14:24:26+08:00
draft: false
---

## 一、组件构成
apiserver 由 3 个组件构成（AggregatorServer、APIServer、APIExtensionServer）

AggregatorServer：实现请求的代理转发，将来自用户的请求拦截转发给其他服务器，并且负责整个 APIServer 的服务发现功能



APIServer：负责对内建资源对象请求的一些处理，包括认证、鉴权等，以及处理各个内建资源的 REST 服务



APIExtensionServer：主要处理自定义资源对象（CR、CRD）的请求
![](https://img-blog.csdnimg.cn/20200420171926724.png)


## 二、程序流程源码分析
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200510165435561.png)
入口函数 

```go
//cmd/kube-apiserver/app/server.go
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	// 完成 server 初始化
	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}
    //PrepareRun：运行前准备（健康检查、存活检查和OpenAPI路由的注册）
    //Run：启动安全的http server提供服务
	return server.PrepareRun().Run(stopCh)
}

```
CreateServerChain

```go
//cmd/kube-apiserver/app/server.go
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*genericapiserver.GenericAPIServer, error) {
	nodeTunneler, proxyTransport, err := CreateNodeDialer(completedOptions)
	if err != nil {
		return nil, err
	}

	kubeAPIServerConfig, sharedInformers, versionedInformers, insecureServingOptions, serviceResolver, pluginInitializer, admissionPostStartHook, err := 
	//创建 KubeAPIServer 所需要的配置（apiServer启动参数配置、分配service的ip、初始化认证授权配置）
	CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	if err != nil {
		return nil, err
	}

	// 判断是否启用了扩展的apiServer ，调用createAPIExtensionsConfig加载扩展apiServer的配置
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, versionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount)
	if err != nil {
		return nil, err
	}
	//创建 apiExtensionsServer 实例
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}
    // 创建 kubeAPIServer 实例
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, sharedInformers, versionedInformers, admissionPostStartHook)
	if err != nil {
		return nil, err
	}
	
	kubeAPIServer.GenericAPIServer.PrepareRun()

	apiExtensionsServer.GenericAPIServer.PrepareRun()

	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, versionedInformers, serviceResolver, proxyTransport, pluginInitializer)
	if err != nil {
		return nil, err
	}
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
	if err != nil {
		return nil, err
	}

	if insecureServingOptions != nil {
		insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(aggregatorServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
		if err := kubeserver.NonBlockingRun(insecureServingOptions, insecureHandlerChain, kubeAPIServerConfig.GenericConfig.RequestTimeout, stopCh); err != nil {
			return nil, err
		}
	}

	return aggregatorServer.GenericAPIServer, nil
}

```
InstallLegacyAPI
```go
//pkg/master/master.go
func (m *Master) InstallLegacyAPI(......) error {
    //NewLegacyRESTStorage创建多种资源的 Storage对象
    legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
    if err != nil {
        return fmt.Errorf("Error building core storage: %v", err)
    }
    ......

    if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
        return fmt.Errorf("Error in registering group versions: %v", err)
    }
    return nil
}
```

通过NewLegacyRESTStorage创建了多种资源的 Storage对象（pod、secret等），Storage保存了资源对象的基本字段信息，也就是apiServer和etcd交互的资源对象数据类型。


```go
//pkg/registry/core/rest/storage_core.go
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
	apiGroupInfo := genericapiserver.APIGroupInfo{
		PrioritizedVersions:          legacyscheme.Scheme.PrioritizedVersionsForGroup(""),
		VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
		Scheme:               legacyscheme.Scheme,
		ParameterCodec:       legacyscheme.ParameterCodec,
		NegotiatedSerializer: legacyscheme.Codecs,
	}

	var podDisruptionClient policyclient.PodDisruptionBudgetsGetter
	if policyGroupVersion := (schema.GroupVersion{Group: "policy", Version: "v1beta1"}); legacyscheme.Scheme.IsVersionRegistered(policyGroupVersion) {
		var err error
		podDisruptionClient, err = policyclient.NewForConfig(c.LoopbackClientConfig)
		if err != nil {
			return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
		}
	}
	restStorage := LegacyRESTStorage{}
    //创建podTemplate的Storage
	podTemplateStorage := podtemplatestore.NewREST(restOptionsGetter)
    //创建event的Storage
	eventStorage := eventstore.NewREST(restOptionsGetter, uint64(c.EventTTL.Seconds()))
	//创建limitRange的Storage
	limitRangeStorage := limitrangestore.NewREST(restOptionsGetter)
    //创建resourceQuota的Storage
	resourceQuotaStorage, resourceQuotaStatusStorage := resourcequotastore.NewREST(restOptionsGetter)
	//创建secret的Storage
	secretStorage := secretstore.NewREST(restOptionsGetter)
	//创建pv的Storage
	persistentVolumeStorage, persistentVolumeStatusStorage := pvstore.NewREST(restOptionsGetter)
	//创建pvc的Storage
	persistentVolumeClaimStorage, persistentVolumeClaimStatusStorage := pvcstore.NewREST(restOptionsGetter)
	//创建configMap的Storage
	configMapStorage := configmapstore.NewREST(restOptionsGetter)
    //创建namespace的Storage
	namespaceStorage, namespaceStatusStorage, namespaceFinalizeStorage := namespacestore.NewREST(restOptionsGetter)
    //创建endpoint的Storage
	endpointsStorage := endpointsstore.NewREST(restOptionsGetter)
    //创建node的Storage
	nodeStorage, err := nodestore.NewStorage(restOptionsGetter, c.KubeletClientConfig, c.ProxyTransport)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
    //创建pod的Storage
	podStorage := podstore.NewStorage(
		restOptionsGetter,
		nodeStorage.KubeletConnectionInfo,
		c.ProxyTransport,
		podDisruptionClient,
	)

	var serviceAccountStorage *serviceaccountstore.REST
	if c.ServiceAccountIssuer != nil && utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) {
		serviceAccountStorage = serviceaccountstore.NewREST(restOptionsGetter, c.ServiceAccountIssuer, c.ServiceAccountAPIAudiences, podStorage.Pod.Store, secretStorage.Store)
	} else {
		serviceAccountStorage = serviceaccountstore.NewREST(restOptionsGetter, nil, nil, nil, nil)
	}

	serviceRESTStorage, serviceStatusStorage := servicestore.NewGenericREST(restOptionsGetter)

	var serviceClusterIPRegistry rangeallocation.RangeRegistry
	serviceClusterIPRange := c.ServiceIPRange
	if serviceClusterIPRange.IP == nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, fmt.Errorf("service clusterIPRange is missing")
	}

	serviceStorageConfig, err := c.StorageFactory.NewConfig(api.Resource("services"))
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	serviceClusterIPAllocator := ipallocator.NewAllocatorCIDRRange(&serviceClusterIPRange, func(max int, rangeSpec string) allocator.Interface {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd := serviceallocator.NewEtcd(mem, "/ranges/serviceips", api.Resource("serviceipallocations"), serviceStorageConfig)
		serviceClusterIPRegistry = etcd
		return etcd
	})
	restStorage.ServiceClusterIPAllocator = serviceClusterIPRegistry

	var serviceNodePortRegistry rangeallocation.RangeRegistry
	serviceNodePortAllocator := portallocator.NewPortAllocatorCustom(c.ServiceNodePortRange, func(max int, rangeSpec string) allocator.Interface {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd := serviceallocator.NewEtcd(mem, "/ranges/servicenodeports", api.Resource("servicenodeportallocations"), serviceStorageConfig)
		serviceNodePortRegistry = etcd
		return etcd
	})
	restStorage.ServiceNodePortAllocator = serviceNodePortRegistry

	controllerStorage := controllerstore.NewStorage(restOptionsGetter)

	serviceRest, serviceRestProxy := servicestore.NewREST(serviceRESTStorage, endpointsStorage, podStorage.Pod, serviceClusterIPAllocator, serviceNodePortAllocator, c.ProxyTransport)
    // 绑定API路由和对应Storage对象
	restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.Binding,

		"podTemplates": podTemplateStorage,

		"replicationControllers":        controllerStorage.Controller,
		"replicationControllers/status": controllerStorage.Status,

		"services":        serviceRest,
		"services/proxy":  serviceRestProxy,
		"services/status": serviceStatusStorage,

		"endpoints": endpointsStorage,

		"nodes":        nodeStorage.Node,
		"nodes/status": nodeStorage.Status,
		"nodes/proxy":  nodeStorage.Proxy,

		"events": eventStorage,

		"limitRanges":                   limitRangeStorage,
		"resourceQuotas":                resourceQuotaStorage,
		"resourceQuotas/status":         resourceQuotaStatusStorage,
		"namespaces":                    namespaceStorage,
		"namespaces/status":             namespaceStatusStorage,
		"namespaces/finalize":           namespaceFinalizeStorage,
		"secrets":                       secretStorage,
		"serviceAccounts":               serviceAccountStorage,
		"persistentVolumes":             persistentVolumeStorage,
		"persistentVolumes/status":      persistentVolumeStatusStorage,
		"persistentVolumeClaims":        persistentVolumeClaimStorage,
		"persistentVolumeClaims/status": persistentVolumeClaimStatusStorage,
		"configMaps":                    configMapStorage,

		"componentStatuses": componentstatus.NewStorage(componentStatusStorage{c.StorageFactory}.serversToValidate),
	}
	if legacyscheme.Scheme.IsVersionRegistered(schema.GroupVersion{Group: "autoscaling", Version: "v1"}) {
		restStorageMap["replicationControllers/scale"] = controllerStorage.Scale
	}
	if legacyscheme.Scheme.IsVersionRegistered(schema.GroupVersion{Group: "policy", Version: "v1beta1"}) {
		restStorageMap["pods/eviction"] = podStorage.Eviction
	}
	if serviceAccountStorage.Token != nil {
		restStorageMap["serviceaccounts/token"] = serviceAccountStorage.Token
	}
	apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap

	return restStorage, apiGroupInfo, nil
}
```
InstallLegacyAPIGroup
 --> installAPIResources （为每一个 API resource 调用 apiGroupVersion.InstallREST 添加路由）
 --> apiGroupVersion.InstallREST （将 restful.WebServic 对象添加到 container 中）
 --> installer.Install（返回最终的 restful.WebService 对象）
 --> registerResourceHandlers（通过go-restful框架实现路由和对应handler处理逻辑的绑定）

```go
//staging/src/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {       
    admit := a.group.Admit

    ......

    // 1、判断该 resource 实现了哪些 REST 操作接口，以此来判断其支持的 verbs 以便为其添加路由
    creater, isCreater := storage.(rest.Creater)
    namedCreater, isNamedCreater := storage.(rest.NamedCreater)
    lister, isLister := storage.(rest.Lister)
    getter, isGetter := storage.(rest.Getter)
    getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
    gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
    collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
    updater, isUpdater := storage.(rest.Updater)
    patcher, isPatcher := storage.(rest.Patcher)
    watcher, isWatcher := storage.(rest.Watcher)
    connecter, isConnecter := storage.(rest.Connecter)
    storageMeta, isMetadata := storage.(rest.StorageMetadata)
    storageVersionProvider, isStorageVersionProvider := storage.(rest.StorageVersionProvider)
    if !isMetadata {
        storageMeta = defaultStorageMetadata{}
    }
    exporter, isExporter := storage.(rest.Exporter)
    if !isExporter {
        exporter = nil
    }

    ......

    // 2、为 resource 添加对应的 actions 并根据是否支持 namespace 
    switch {
    case !namespaceScoped:
        ......

        actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
        actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
        actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
        actions = appendIf(actions, action{"WATCHLIST", "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

        actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
        if getSubpath {
            actions = appendIf(actions, action{"GET", itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
        }
        actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
        actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
        actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)
        actions = appendIf(actions, action{"WATCH", "watch/" + itemPath, nameParams, namer, false}, isWatcher)
        actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
        actions = appendIf(actions, action{"CONNECT", itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)
    default:
        ......
        actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
        actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
        actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
        actions = appendIf(actions, action{"WATCHLIST", "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

        actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
        ......
    }

    // 3、根据 action 创建对应的 route
    kubeVerbs := map[string]struct{}{}
    reqScope := handlers.RequestScope{
        Serializer:      a.group.Serializer,
        ParameterCodec:  a.group.ParameterCodec,
        Creater:         a.group.Creater,
        Convertor:       a.group.Convertor,
        ......
    }
    ......
    // 4、从 rest.Storage 到 restful.Route 映射
    // 为每个操作添加对应的 handler
    for _, action := range actions {
        ......
        verbOverrider, needOverride := storage.(StorageMetricsOverride)
        switch action.Verb {
        case "GET": ......
        case "LIST":
        case "PUT":
        case "PATCH":
        // 此处以 POST 操作进行说明
        case "POST": 
            var handler restful.RouteFunction
            // 5、初始化 handler
            if isNamedCreater {
                handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
            } else {
                handler = restfulCreateResource(creater, reqScope, admit)
            }
            handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, handler)
            article := GetArticleForNoun(kind, " ")
            doc := "create" + article + kind
            if isSubresource {
                doc = "create " + subresource + " of" + article + kind
            }
            // 6、route 与 handler 进行绑定
            route := ws.POST(action.Path).To(handler).
                Doc(doc).
                Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
                Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).
                Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
                Returns(http.StatusOK, "OK", producedObject).
                Returns(http.StatusCreated, "Created", producedObject).
                Returns(http.StatusAccepted, "Accepted", producedObject).
                Reads(defaultVersionedObject).
                Writes(producedObject)
            if err := AddObjectParams(ws, route, versionedCreateOptions); err != nil {
                return nil, err
            }
            addParams(route, action.Params)
            // 7、添加到路由中
            routes = append(routes, route)
        case "DELETE": 
        case "DELETECOLLECTION":
        case "WATCH":
        case "WATCHLIST":
        case "CONNECT":
        default:
    }
    ......
    return &apiResource, nil
}
```
createHandler 与etcd交互流程

```go
//staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go
func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
    return func(w http.ResponseWriter, req *http.Request) {
        trace := utiltrace.New("Create", utiltrace.Field{"url", req.URL.Path})
        defer trace.LogIfLong(500 * time.Millisecond)
        ......

        gv := scope.Kind.GroupVersion()
        // 1、得到合适的SerializerInfo
        s, err := negotiation.NegotiateInputSerializer(req, false, scope.Serializer)
        if err != nil {
            scope.err(err, w, req)
            return
        }
        // 2、找到合适的 decoder
        decoder := scope.Serializer.DecoderToVersion(s.Serializer, scope.HubGroupVersion)

        body, err := limitedReadBody(req, scope.MaxRequestBodyBytes)
        if err != nil {
            scope.err(err, w, req)
            return
        }

        ......

        defaultGVK := scope.Kind
        original := r.New()
        trace.Step("About to convert to expected version")
        // 3、decoder 解码
        obj, gvk, err := decoder.Decode(body, &defaultGVK, original)
        ......

        ae := request.AuditEventFrom(ctx)
        admit = admission.WithAudit(admit, ae)
        audit.LogRequestObject(ae, obj, scope.Resource, scope.Subresource, scope.Serializer)

        userInfo, _ := request.UserFrom(ctx)


        if len(name) == 0 {
            _, name, _ = scope.Namer.ObjectName(obj)
        }
        // 4、执行 kube-apiserver 启动时加载的 admission-plugins（webhook、validation）
        admissionAttributes := admission.NewAttributesRecord(......)
        if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) {
            err = mutatingAdmission.Admit(ctx, admissionAttributes, scope)
            if err != nil {
                scope.err(err, w, req)
                return
            }
        }

        ......
        // 5、存入etcd（Create方法由每个Storage对象自己实现）
        result, err := finishRequest(timeout, func() (runtime.Object, error) {
            return r.Create(
                ctx,
                name,
                obj,
                rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope),
                options,
            )
        })
        ......
    }
}
```
## 三、客户端访问ApiServer流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020051017160293.png)