# 【K8s源码品读】006：Phase 1 - kube-apiserver - GenericAPIServer的初始化

## 聚焦目标

理解kube-apiserver是中的管理核心资源的`KubeAPIServer`是怎么启动的



## 目录

1. [genericServer的创建](#genericServer)
2. [创建REST的Handler](#NewAPIServerHandler)
3. [Generic的API路由规则](#installAPI)
4. [初始化核心Apiserver](#apiserver)
5. [核心资源的API路由规则](#InstallLegacyAPI)
6. [创建Pod的函数](#create-pod)





## GenericServer

```go
// 在APIExtensionsServer、KubeAPIServer和AggregatorServer三种Server启动时，我们都能发现这么一个函数
// APIExtensionsServer
genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
// KubeAPIServer
s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
// AggregatorServer
genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)

// 都通过GenericConfig创建了genericServer，我们先大致浏览下
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
	// 新建Handler
	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
  
	// 实例化一个Server
	s := &GenericAPIServer{
    ...
  }

	// 处理钩子hook操作
	for k, v := range delegationTarget.PostStartHooks() {
		s.postStartHooks[k] = v
	}

	for k, v := range delegationTarget.PreShutdownHooks() {
		s.preShutdownHooks[k] = v
	}

	// 健康监测
	for _, delegateCheck := range delegationTarget.HealthzChecks() {
		skip := false
		for _, existingCheck := range c.HealthzChecks {
			if existingCheck.Name() == delegateCheck.Name() {
				skip = true
				break
			}
		}
		if skip {
			continue
		}
		s.AddHealthChecks(delegateCheck)
	}
	
  // 安装API相关参数，这个是重点
	installAPI(s, c.Config)

	return s, nil
}
```



## NewAPIServerHandler

```go
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	// 采用了 github.com/emicklei/go-restful 这个库作为 RESTful 接口的设计，目前了解即可
	gorestfulContainer := restful.NewContainer()
	
}
```



## installAPI

```go
func installAPI(s *GenericAPIServer, c *Config) {
  // 添加 /index.html 路由规则
	if c.EnableIndex {
		routes.Index{}.Install(s.listedPathProvider, s.Handler.NonGoRestfulMux)
	}
  // 添加go语言 /pprof 的路由规则，常用于性能分析
	if c.EnableProfiling {
		routes.Profiling{}.Install(s.Handler.NonGoRestfulMux)
		if c.EnableContentionProfiling {
			goruntime.SetBlockProfileRate(1)
		}
		routes.DebugFlags{}.Install(s.Handler.NonGoRestfulMux, "v", routes.StringFlagPutHandler(logs.GlogSetter))
	}
  // 添加监控相关的 /metrics 的指标路由规则
	if c.EnableMetrics {
		if c.EnableProfiling {
			routes.MetricsWithReset{}.Install(s.Handler.NonGoRestfulMux)
		} else {
			routes.DefaultMetrics{}.Install(s.Handler.NonGoRestfulMux)
		}
	}
	// 添加版本 /version 的路由规则
	routes.Version{Version: c.Version}.Install(s.Handler.GoRestfulContainer)
	// 开启服务发现
	if c.EnableDiscovery {
		s.Handler.GoRestfulContainer.Add(s.DiscoveryGroupManager.WebService())
	}
	if feature.DefaultFeatureGate.Enabled(features.APIPriorityAndFairness) {
		c.FlowControl.Install(s.Handler.NonGoRestfulMux)
	}
}
```



## Apiserver

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {
	// genericServer的初始化
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	// 核心KubeAPIServer的实例化
	m := &Master{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

	// 注册Legacy API的注册
	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{}
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}
	// REST接口的存储定义，可以看到很多k8s上的常见定义，比如node节点/storage存储/event事件等等
	restStorageProviders := []RESTStorageProvider{
		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		discoveryrest.StorageProvider{},
		extensionsrest.RESTStorageProvider{},
		networkingrest.RESTStorageProvider{},
		noderest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		settingsrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.StorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
  // 注册API
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}
	// 添加Hook
	m.GenericAPIServer.AddPostStartHookOrDie("start-cluster-authentication-info-controller", func(hookContext genericapiserver.PostStartHookContext) error {
	})
	return m, nil
}
```

注册API的关键在`InstallLegacyAPI`和`InstallAPIs`，如果你对kubernetes的资源有一定的了解，会知道核心资源都放在Legacy中（如果不了解的话，点击函数看一下，就能有所有了解）



## InstallLegacyAPI

```go
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) error {
  // RESTStorage的初始化
	legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
  
  // 前缀为 /api，注册上对应的Version和Resource
  // Pod作为核心资源，没有Group的概念
	if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}

// 我们再细看这个RESTStorage的初始化
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
	// pod 模板
	podTemplateStorage, err := podtemplatestore.NewREST(restOptionsGetter)
	// event事件
	eventStorage, err := eventstore.NewREST(restOptionsGetter, uint64(c.EventTTL.Seconds()))
	// limitRange资源限制
	limitRangeStorage, err := limitrangestore.NewREST(restOptionsGetter)
	// resourceQuota资源配额
	resourceQuotaStorage, resourceQuotaStatusStorage, err := resourcequotastore.NewREST(restOptionsGetter)
	// secret加密
	secretStorage, err := secretstore.NewREST(restOptionsGetter)
	// PV 存储
	persistentVolumeStorage, persistentVolumeStatusStorage, err := pvstore.NewREST(restOptionsGetter)
	// PVC 存储
	persistentVolumeClaimStorage, persistentVolumeClaimStatusStorage, err := pvcstore.NewREST(restOptionsGetter)
	// ConfigMap 配置
	configMapStorage, err := configmapstore.NewREST(restOptionsGetter)
	// 等等核心资源，暂不一一列举
  
  // pod模板，我们的示例nginx-pod属于这个类型的资源
  podStorage, err := podstore.NewStorage()
 
  // 保存storage的对应关系
  restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.LegacyBinding,
    ...
  }
}
```



## Create Pod

```go
// 查看Pod初始化
func NewStorage(optsGetter generic.RESTOptionsGetter, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper, podDisruptionBudgetClient policyclient.PodDisruptionBudgetsGetter) (PodStorage, error) {

	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &api.Pod{} },
		NewListFunc:              func() runtime.Object { return &api.PodList{} },
		PredicateFunc:            registrypod.MatchPod,
		DefaultQualifiedResource: api.Resource("pods"),
		// 增改删的策略
		CreateStrategy:      registrypod.Strategy,
		UpdateStrategy:      registrypod.Strategy,
		DeleteStrategy:      registrypod.Strategy,
		ReturnDeletedObject: true,

		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
	}
}
// 查看 Strategy 的初始化
var Strategy = podStrategy{legacyscheme.Scheme, names.SimpleNameGenerator}

// 又查询到Scheme的初始化。Schema可以理解为Kubernetes的注册表，即所有的资源类型必须先注册进Schema才可使用
var Scheme = runtime.NewScheme()
```

