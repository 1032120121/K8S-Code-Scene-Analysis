
ApiServer中一个资源变更请求要经历的完整流程主要包括认证(Authentication)，授权(Autoritation), 审计（Audit）, 准入（Admission）,GVK转换，对象持久化

注入插件的配置
--enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
--disable-admission-plugins=PodNodeSelector,AlwaysDeny ...

TODO：三个apiserver的关系

gvk转换
```Golang

KubeAPIServer
	encodeVersioner := runtime.NewMultiGroupVersioner(
		opts.StorageVersion,
		schema.GroupKind{Group: opts.StorageVersion.Group},
		schema.GroupKind{Group: opts.MemoryVersion.Group},
	)
APIExtensionsServer
etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1beta1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
```

# 实现
为了完整分析一个Server的请求处理流程，先忽略Server链中另外的ApiExtensionsServer和AggregatorServer。下面只以普通的KubeApiserver为例
## 构建服务配置
目标是构建配置核心结构GenericAPIServer的genericapiserver.Config对象，它包含了整个server处理流程中的主要结构
```Golang
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
	kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions)
	// XXX
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
	// XXX
	return aggregatorServer, nil
}

// CreateKubeAPIServerConfig creates all the resources for running the API server, but runs none of them
func CreateKubeAPIServerConfig(s completedServerRunOptions) (
	*controlplane.Config,
	aggregatorapiserver.ServiceResolver,
	[]admission.PluginInitializer,
	error,
) {
	// XXX
	genericConfig, versionedInformers, serviceResolver, pluginInitializers, admissionPostStartHook, storageFactory, err := buildGenericConfig(s.ServerRunOptions, proxyTransport)
	// XXX
	config := &controlplane.Config{
		GenericConfig: genericConfig,
		// XXX
	}
	return config, serviceResolver, pluginInitializers, nil
}

// Config is a structure used to configure a GenericAPIServer.
// Its members are sorted roughly in order of importance for composers.
type Config struct {
	// SecureServing is required to serve https
	SecureServing *SecureServingInfo

	// Authentication is the configuration for authentication
	Authentication AuthenticationInfo

	// Authorization is the configuration for authorization
	Authorization AuthorizationInfo

	// LoopbackClientConfig is a config for a privileged loopback connection to the API server
	// This is required for proper functioning of the PostStartHooks on a GenericAPIServer
	// TODO: move into SecureServing(WithLoopback) as soon as insecure serving is gone
	LoopbackClientConfig *restclient.Config

	// EgressSelector provides a lookup mechanism for dialing outbound connections.
	// It does so based on a EgressSelectorConfiguration which was read at startup.
	EgressSelector *egressselector.EgressSelector

	// RuleResolver is required to get the list of rules that apply to a given user
	// in a given namespace
	RuleResolver authorizer.RuleResolver
	// AdmissionControl performs deep inspection of a given request (including content)
	// to set values and determine whether its allowed
	AdmissionControl      admission.Interface
	CorsAllowedOriginList []string
	HSTSDirectives        []string
	// FlowControl, if not nil, gives priority and fairness to request handling
	FlowControl utilflowcontrol.Interface

	EnableIndex     bool
	EnableProfiling bool
	EnableDiscovery bool
	// Requires generic profiling enabled
	EnableContentionProfiling bool
	EnableMetrics             bool

	DisabledPostStartHooks sets.String
	// done values in this values for this map are ignored.
	PostStartHooks map[string]PostStartHookConfigEntry

	// Version will enable the /version endpoint if non-nil
	Version *version.Info
	// AuditBackend is where audit events are sent to.
	AuditBackend audit.Backend
	// AuditPolicyChecker makes the decision of whether and how to audit log a request.
	AuditPolicyChecker auditpolicy.Checker
	// ExternalAddress is the host name to use for external (public internet) facing URLs (e.g. Swagger)
	// Will default to a value based on secure serving info and available ipv4 IPs.
	ExternalAddress string

	// TracerProvider can provide a tracer, which records spans for distributed tracing.
	TracerProvider *trace.TracerProvider
	
	//===========================================================================
	// Fields you probably don't care about changing
	//===========================================================================
	
	// BuildHandlerChainFunc allows you to build custom handler chains by decorating the apiHandler.
	BuildHandlerChainFunc func(apiHandler http.Handler, c *Config) (secure http.Handler)
	// XXX
	// RESTOptionsGetter is used to construct RESTStorage types via the generic registry.
	RESTOptionsGetter genericregistry.RESTOptionsGetter
}

// 关键的配置函数，整个函数都是在做genericConfig的配置
// BuildGenericConfig takes the master server options and produces the genericapiserver.Config associated with it
func buildGenericConfig(
	s *options.ServerRunOptions,
	proxyTransport *http.Transport,
) (
	genericConfig *genericapiserver.Config,   // 关键Config
	versionedInformers clientgoinformers.SharedInformerFactory,
	serviceResolver aggregatorapiserver.ServiceResolver,
	pluginInitializers []admission.PluginInitializer,
	admissionPostStartHook genericapiserver.PostStartHookFunc,
	storageFactory *serverstorage.DefaultStorageFactory,
	lastErr error,
) {
    genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
    genericConfig.MergedResourceConfig = controlplane.DefaultAPIResourceConfigSource()
    
    // 以下是各种ApplyTo是将命令行启动的选型配置赋值给genericConfig
    if lastErr = s.GenericServerRunOptions.ApplyTo(genericConfig); lastErr != nil {
	return
    }
    // XXX
    // 配置genericConfig的MergedResourceConfig, 最终决定支持哪些GroupVersion，对应的Resource是否开启使能
    if lastErr = s.APIEnablement.ApplyTo(genericConfig, controlplane.DefaultAPIResourceConfigSource(), legacyscheme.Scheme); lastErr != nil {
	return
    }
  
    // XXX
    // 配置存储内部特定的Resource和version，如csistoragecapacities/v1beta1。存储的gvr和外部api的gvr关注点不同，所以是分开的。
    storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()
    storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig
    // etcd的参数配置到storageFactoryConfig
    completedStorageFactoryConfig, err := storageFactoryConfig.Complete(s.Etcd) 
    if err != nil {
	lastErr = err
	return
    }
    // 决定同一种Resource的不同Group选择哪个作为存储位置，比如apps.deployments和extensions.deployments
    storageFactory, lastErr = completedStorageFactoryConfig.New()  
    if lastErr != nil {
	return
    }  
    // XX
    // 设置RESTOptionsGetter，它决定各个Resource的RestStorage
    if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil { 
	return
    }
}
```
下面针对不同的流程来分别介绍buildGenericConfig的细节。

## 完整的http处理流程

buildGenericConfig首先定义了http handler链DefaultBuildHandlerChain
```Golang
// NewConfig returns a Config struct with the default values
func NewConfig(codecs serializer.CodecFactory) *Config {
	// XXX
	return &Config{
		Serializer:                  codecs,
		BuildHandlerChainFunc:       DefaultBuildHandlerChain,  // http handler链
		HandlerChainWaitGroup:       new(utilwaitgroup.SafeWaitGroup),
		LegacyAPIGroupPrefixes:      sets.NewString(DefaultLegacyAPIPrefix),
		DisabledPostStartHooks:      sets.NewString(),
		PostStartHooks:              map[string]PostStartHookConfigEntry{},
		HealthzChecks:               append([]healthz.HealthChecker{}, defaultHealthChecks...),
		ReadyzChecks:                append([]healthz.HealthChecker{}, defaultHealthChecks...),
		LivezChecks:                 append([]healthz.HealthChecker{}, defaultHealthChecks...),
		EnableIndex:                 true,
		EnableDiscovery:             true,
		EnableProfiling:             true,
		EnableMetrics:               true,
		MaxRequestsInFlight:         400,
		MaxMutatingRequestsInFlight: 200,
		RequestTimeout:              time.Duration(60) * time.Second,
		MinRequestTimeout:           1800,
		LivezGracePeriod:            time.Duration(0),
		ShutdownDelayDuration:       time.Duration(0),
		// 1.5MB is the default client request size in bytes
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest size
		// increase the "copy" operations in a json patch may cause.
		JSONPatchMaxCopyBytes: int64(3 * 1024 * 1024),
		// 1.5MB is the recommended client request size in byte
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest request
		// body size to be accepted and decoded in a write request.
		MaxRequestBodyBytes: int64(3 * 1024 * 1024),

		// Default to treating watch as a long-running operation
		// Generic API servers have no inherent long-running subresources
		LongRunningFunc:       genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"), sets.NewString()),
		RequestWidthEstimator: flowcontrolrequest.DefaultWidthEstimator,
		lifecycleSignals:      newLifecycleSignals(),

		APIServerID:           id,
		StorageVersionManager: storageversion.NewDefaultManager(),
	}
}

// handler处理的顺序和此函数中定义的顺序一致
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	// 授权
	handler = genericapifilters.WithAuthorization(handler, c.Authorization.Authorizer, c.Serializer)
	
	// 流控
	if c.FlowControl != nil {
		handler = filterlatency.TrackCompleted(handler)
		handler = genericfilters.WithPriorityAndFairness(handler, c.LongRunningFunc, c.FlowControl, c.RequestWidthEstimator)
		handler = filterlatency.TrackStarted(handler, "priorityandfairness")
	} else {
		handler = genericfilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
	}
	
	// 用户冒充检查，一个老user可以在http header中携带Impersonate-User、Impersonate-Group、Impersonate-Uid、Impersonate-Extra-信息以新的user身份做请求
	handler = genericapifilters.WithImpersonation(handler, c.Authorization.Authorizer, c.Serializer)

	// 审计
	handler = genericapifilters.WithAudit(handler, c.AuditBackend, c.AuditPolicyChecker, c.LongRunningFunc)
	failedHandler = genericapifilters.WithFailedAuthenticationAudit(failedHandler, c.AuditBackend, c.AuditPolicyChecker)

        // 认证
	handler = genericapifilters.WithAuthentication(handler, c.Authentication.Authenticator, failedHandler, c.Authentication.APIAudiences)

        // https://github.com/martini-contrib/cors
	handler = genericfilters.WithCORS(handler, c.CorsAllowedOriginList, nil, nil, nil, "true")

	// WithTimeoutForNonLongRunningRequests will call the rest of the request handling in a go-routine with the
	// context with deadline. The go-routine can keep running, while the timeout logic will return a timeout to the client.
	handler = genericfilters.WithTimeoutForNonLongRunningRequests(handler, c.LongRunningFunc)

	handler = genericapifilters.WithRequestDeadline(handler, c.AuditBackend, c.AuditPolicyChecker,
		c.LongRunningFunc, c.Serializer, c.RequestTimeout)
	handler = genericfilters.WithWaitGroup(handler, c.LongRunningFunc, c.HandlerChainWaitGroup)
	if c.SecureServing != nil && !c.SecureServing.DisableHTTP2 && c.GoawayChance > 0 {
		handler = genericfilters.WithProbabilisticGoaway(handler, c.GoawayChance)
	}
	handler = genericapifilters.WithAuditAnnotations(handler, c.AuditBackend, c.AuditPolicyChecker)
	handler = genericapifilters.WithWarningRecorder(handler)
	handler = genericapifilters.WithCacheControl(handler)
	handler = genericfilters.WithHSTS(handler, c.HSTSDirectives)
	handler = genericfilters.WithHTTPLogging(handler)
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		handler = genericapifilters.WithTracing(handler, c.TracerProvider)
	}
	handler = genericapifilters.WithRequestInfo(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithRequestReceivedTimestamp(handler)
	handler = genericfilters.WithPanicRecovery(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithAuditID(handler)
	return handler
}
```

## ETCD存储

RESTOptionsGetter接口由StorageFactoryRestOptionsFactory对象实现，主要就是对每种Resource返回自己的后端存储ETCD配置。
```Golang
type RESTOptionsGetter interface {
	GetRESTOptions(resource schema.GroupResource) (RESTOptions, error)
}

type StorageFactoryRestOptionsFactory struct {
	// XXX
	StorageFactory serverstorage.StorageFactory  // 在GVR版本转换流程中介绍
}

func (f *StorageFactoryRestOptionsFactory) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {
        // 确定该Resource对后端ETCD的详细配置，如访问ETCD序列化参数，media type，编解码器，在持久化和内存中不同的GroupVersion表示对象等
	storageConfig, err := f.StorageFactory.NewConfig(resource) 
	if err != nil {
		return generic.RESTOptions{}, fmt.Errorf("unable to find storage destination for %v, due to %v", resource, err.Error())
	}

	ret := generic.RESTOptions{
		StorageConfig:           storageConfig,
		Decorator:               generic.UndecoratedStorage, // 增删查改后端etcd中对象的工厂方法
		DeleteCollectionWorkers: f.Options.DeleteCollectionWorkers,
		EnableGarbageCollection: f.Options.EnableGarbageCollection,
		ResourcePrefix:          f.StorageFactory.ResourcePrefix(resource),
		CountMetricPollPeriod:   f.Options.StorageConfig.CountMetricPollPeriod,
	}
	if f.Options.EnableWatchCache {
		sizes, err := ParseWatchCacheSizes(f.Options.WatchCacheSizes)
		if err != nil {
			return generic.RESTOptions{}, err
		}
		size, ok := sizes[resource]
		if ok && size > 0 {
			klog.Warningf("Dropping watch-cache-size for %v - watchCache size is now dynamic", resource)
		}
		if ok && size <= 0 {
		        // 原始的无装饰器的工厂方法是etcdV3 client的直接实现
			ret.Decorator = generic.UndecoratedStorage
		} else {
		        // 带cache装饰器的工厂方法是带有List&Watch机制的，参见另一篇文章：apiserver的cache内部原理
			ret.Decorator = genericregistry.StorageWithCacher()
		}
	}

	return ret, nil
}
```

RestOptions的装饰器函数目的是返回针对后端存储ETCD的增删查改接口，其中带cache的装饰器对象Cacher也实现了该存储接口，对读接口Watch/Get等进行cache处理
```Golang
// StorageDecorator is a function signature for producing a storage.Interface
// and an associated DestroyFunc from given parameters.
type StorageDecorator func(
	config *storagebackend.Config,
	resourcePrefix string,
	keyFunc func(obj runtime.Object) (string, error),
	newFunc func() runtime.Object,
	newListFunc func() runtime.Object,
	getAttrsFunc storage.AttrFunc,
	trigger storage.IndexerFuncs,
	indexers *cache.Indexers) (storage.Interface, factory.DestroyFunc, error)

// file: vendor/k8s\.io/apiserver/pkg/storage/inteface.go
// Interface offers a common interface for object marshaling/unmarshaling operations and
// hides all the storage-related operations behind it.
type Interface interface {
	// Returns Versioner associated with this interface.
	Versioner() Versioner

	// Create adds a new object at a key unless it already exists. 'ttl' is time-to-live
	// in seconds (0 means forever). If no error is returned and out is not nil, out will be
	// set to the read value from database.
	Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error

	// Delete removes the specified key and returns the value that existed at that spot.
	// If key didn't exist, it will return NotFound storage error.
	// If 'cachedExistingObject' is non-nil, it can be used as a suggestion about the
	// current version of the object to avoid read operation from storage to get it.
	// However, the implementations have to retry in case suggestion is stale.
	Delete(
		ctx context.Context, key string, out runtime.Object, preconditions *Preconditions,
		validateDeletion ValidateObjectFunc, cachedExistingObject runtime.Object) error

	// Watch begins watching the specified key. Events are decoded into API objects,
	// and any items selected by 'p' are sent down to returned watch.Interface.
	// resourceVersion may be used to specify what version to begin watching,
	// which should be the current resourceVersion, and no longer rv+1
	// (e.g. reconnecting without missing any updates).
	// If resource version is "0", this interface will get current object at given key
	// and send it in an "ADDED" event, before watch starts.
	Watch(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)

	// WatchList begins watching the specified key's items. Items are decoded into API
	// objects and any item selected by 'p' are sent down to returned watch.Interface.
	// resourceVersion may be used to specify what version to begin watching,
	// which should be the current resourceVersion, and no longer rv+1
	// (e.g. reconnecting without missing any updates).
	// If resource version is "0", this interface will list current objects directory defined by key
	// and send them in "ADDED" events, before watch starts.
	WatchList(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)

	// Get unmarshals json found at key into objPtr. On a not found error, will either
	// return a zero object of the requested type, or an error, depending on 'opts.ignoreNotFound'.
	// Treats empty responses and nil response nodes exactly like a not found error.
	// The returned contents may be delayed, but it is guaranteed that they will
	// match 'opts.ResourceVersion' according 'opts.ResourceVersionMatch'.
	Get(ctx context.Context, key string, opts GetOptions, objPtr runtime.Object) error

	// GetToList unmarshals json found at key and opaque it into *List api object
	// (an object that satisfies the runtime.IsList definition).
	// The returned contents may be delayed, but it is guaranteed that they will
	// match 'opts.ResourceVersion' according 'opts.ResourceVersionMatch'.
	GetToList(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error

	// List unmarshalls jsons found at directory defined by key and opaque them
	// into *List api object (an object that satisfies runtime.IsList definition).
	// The returned contents may be delayed, but it is guaranteed that they will
	// match 'opts.ResourceVersion' according 'opts.ResourceVersionMatch'.
	List(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error

	// GuaranteedUpdate keeps calling 'tryUpdate()' to update key 'key' (of type 'ptrToType')
	// retrying the update until success if there is index conflict.
	// Note that object passed to tryUpdate may change across invocations of tryUpdate() if
	// other writers are simultaneously updating it, so tryUpdate() needs to take into account
	// the current contents of the object when deciding how the update object should look.
	// If the key doesn't exist, it will return NotFound storage error if ignoreNotFound=false
	// or zero value in 'ptrToType' parameter otherwise.
	// If the object to update has the same value as previous, it won't do any update
	// but will return the object in 'ptrToType' parameter.
	// If 'cachedExistingObject' is non-nil, it can be used as a suggestion about the
	// current version of the object to avoid read operation from storage to get it.
	// However, the implementations have to retry in case suggestion is stale.
	//
	// Example:
	//
	// s := /* implementation of Interface */
	// err := s.GuaranteedUpdate(
	//     "myKey", &MyType{}, true,
	//     func(input runtime.Object, res ResponseMeta) (runtime.Object, *uint64, error) {
	//       // Before each invocation of the user defined function, "input" is reset to
	//       // current contents for "myKey" in database.
	//       curr := input.(*MyType)  // Guaranteed to succeed.
	//
	//       // Make the modification
	//       curr.Counter++
	//
	//       // Return the modified object - return an error to stop iterating. Return
	//       // a uint64 to alter the TTL on the object, or nil to keep it the same value.
	//       return cur, nil, nil
	//    },
	// )
	GuaranteedUpdate(
		ctx context.Context, key string, ptrToType runtime.Object, ignoreNotFound bool,
		preconditions *Preconditions, tryUpdate UpdateFunc, cachedExistingObject runtime.Object) error

	// Count returns number of different entries under the key (generally being path prefix).
	Count(key string) (int64, error)
}
```


## GVR版本处理

### Api全局初始化

在apiserver的起始文件cmd/kube-apiserver/app/server.go中，通过import方式做了大量Group和Version的隐含初始化工作。
```Golang
import （
	"k8s.io/kubernetes/pkg/api/legacyscheme"
	"k8s.io/kubernetes/pkg/controlplane" 
        // xxx
）

```

其中包"k8s.io/kubernetes/pkg/controlplane" 下初始化各个API Group自己的Scheme，注册Group所属的对象类型进去
```Golang
// file: "k8s.io/kubernetes/pkg/controlplane/import_known_version.go"
package controlplane

import (
	// These imports are the API groups the API server will support.
	_ "k8s.io/kubernetes/pkg/apis/admission/install"
	_ "k8s.io/kubernetes/pkg/apis/admissionregistration/install"
	_ "k8s.io/kubernetes/pkg/apis/apiserverinternal/install"
	_ "k8s.io/kubernetes/pkg/apis/apps/install"
	_ "k8s.io/kubernetes/pkg/apis/authentication/install"
	_ "k8s.io/kubernetes/pkg/apis/authorization/install"
	_ "k8s.io/kubernetes/pkg/apis/autoscaling/install"
	_ "k8s.io/kubernetes/pkg/apis/batch/install"
	_ "k8s.io/kubernetes/pkg/apis/certificates/install"
	_ "k8s.io/kubernetes/pkg/apis/coordination/install"
	_ "k8s.io/kubernetes/pkg/apis/core/install"
	_ "k8s.io/kubernetes/pkg/apis/discovery/install"
	_ "k8s.io/kubernetes/pkg/apis/events/install"
	_ "k8s.io/kubernetes/pkg/apis/extensions/install"
	_ "k8s.io/kubernetes/pkg/apis/flowcontrol/install"
	_ "k8s.io/kubernetes/pkg/apis/imagepolicy/install"
	_ "k8s.io/kubernetes/pkg/apis/networking/install"
	_ "k8s.io/kubernetes/pkg/apis/node/install"
	_ "k8s.io/kubernetes/pkg/apis/policy/install"
	_ "k8s.io/kubernetes/pkg/apis/rbac/install"
	_ "k8s.io/kubernetes/pkg/apis/scheduling/install"
	_ "k8s.io/kubernetes/pkg/apis/storage/install"
)

// 以legacy核心api为例
// file: "k8s.io/kubernetes/pkg/apis/core/install/install.go"
func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(core.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion))
}

func addKnownTypes(scheme *runtime.Scheme) error {
	if err := scheme.AddIgnoredConversionType(&metav1.TypeMeta{}, &metav1.TypeMeta{}); err != nil {
		return err
	} 
        // 注册legacy核心api的资源对象
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Pod{},
		&PodList{},
		&PodStatusResult{},
		&PodTemplate{},
		&PodTemplateList{},
		&ReplicationControllerList{},
		&ReplicationController{},
		&ServiceList{},
		&Service{},
		&ServiceProxyOptions{},
		&NodeList{},
		&Node{},
		&NodeProxyOptions{},
		&Endpoints{},
		&EndpointsList{},
		&Binding{},
		&Event{},
		&EventList{},
		&List{},
		&LimitRange{},
		&LimitRangeList{},
		&ResourceQuota{},
		&ResourceQuotaList{},
		&Namespace{},
		&NamespaceList{},
		&ServiceAccount{},
		&ServiceAccountList{},
		&Secret{},
		&SecretList{},
		&PersistentVolume{},
		&PersistentVolumeList{},
		&PersistentVolumeClaim{},
		&PersistentVolumeClaimList{},
		&PodAttachOptions{},
		&PodLogOptions{},
		&PodExecOptions{},
		&PodPortForwardOptions{},
		&PodProxyOptions{},
		&ComponentStatus{},
		&ComponentStatusList{},
		&SerializedReference{},
		&RangeAllocation{},
		&ConfigMap{},
		&ConfigMapList{},
	)

	return nil
}
```
完成隐含初始化后，各个api包下的Scheme、Codecs等全局变量就有值了，如包"k8s.io/kubernetes/pkg/api/legacyscheme" 的legacyscheme.Scheme和legacyscheme.Codecs

### apiserver资源版本的自定义配置
apiserver可以通过--runtime-config启动时参数控制api/all、api/ga、api/beta、api/alpha四种内置API的启用和禁止（一般用在aggregatorServer，普通kubeApiserver不需要）。先确定了server**默认**支持的gv和gvr

### 默认资源版本
再次回到buildGenericConfig。 APIEnablement.ApplyTo函数最终Merge了全局初始化的资源API、自定义资源版本、默认支持的资源版本。自定义资源版本可以覆盖默认的，排除掉未注册到全局Scheme中的，最终决策出要存储到ETCD的GVR
``` Golang
func buildGenericConfig(
    // XXX
    if lastErr = s.APIEnablement.ApplyTo(genericConfig, controlplane.DefaultAPIResourceConfigSource(), legacyscheme.Scheme); lastErr != nil {
	return
    }
    // XXX
}

// ApplyTo override MergedResourceConfig with defaults and registry
func (s *APIEnablementOptions) ApplyTo(c *server.Config, defaultResourceConfig *serverstore.ResourceConfig, registry resourceconfig.GroupVersionRegistry) error {
	// XXX
	// defaultResourceConfig代表默认资源版本；s.RuntimeConfig代表启动时自定义配置资源版本；registry代表全局初始化资源Api
	mergedResourceConfig, err := resourceconfig.MergeAPIResourceConfigs(defaultResourceConfig, s.RuntimeConfig, registry)
	// 设置了genericConfig的MergedResourceConfig
	c.MergedResourceConfig = mergedResourceConfig
	// XXX
}

// DefaultAPIResourceConfigSource returns default configuration for an APIResource.
func DefaultAPIResourceConfigSource() *serverstorage.ResourceConfig {
	ret := serverstorage.NewResourceConfig()
	// NOTE: GroupVersions listed here will be enabled by default. Don't put alpha versions in the list.
	ret.EnableVersions(
		admissionregistrationv1.SchemeGroupVersion,
		admissionregistrationv1beta1.SchemeGroupVersion,
		apiv1.SchemeGroupVersion,
		appsv1.SchemeGroupVersion,
		authenticationv1.SchemeGroupVersion,
		authenticationv1beta1.SchemeGroupVersion,
		authorizationapiv1.SchemeGroupVersion,
		authorizationapiv1beta1.SchemeGroupVersion,
		autoscalingapiv1.SchemeGroupVersion,
		autoscalingapiv2beta1.SchemeGroupVersion,
		autoscalingapiv2beta2.SchemeGroupVersion,
		batchapiv1.SchemeGroupVersion,
		batchapiv1beta1.SchemeGroupVersion,
		certificatesapiv1.SchemeGroupVersion,
		certificatesapiv1beta1.SchemeGroupVersion,
		coordinationapiv1.SchemeGroupVersion,
		coordinationapiv1beta1.SchemeGroupVersion,
		discoveryv1.SchemeGroupVersion,
		discoveryv1beta1.SchemeGroupVersion,
		eventsv1.SchemeGroupVersion,
		eventsv1beta1.SchemeGroupVersion,
		extensionsapiv1beta1.SchemeGroupVersion,
		networkingapiv1.SchemeGroupVersion,
		networkingapiv1beta1.SchemeGroupVersion,
		nodev1.SchemeGroupVersion,
		nodev1beta1.SchemeGroupVersion,
		policyapiv1.SchemeGroupVersion,
		policyapiv1beta1.SchemeGroupVersion,
		rbacv1.SchemeGroupVersion,
		rbacv1beta1.SchemeGroupVersion,
		storageapiv1.SchemeGroupVersion,
		storageapiv1beta1.SchemeGroupVersion,
		schedulingapiv1beta1.SchemeGroupVersion,
		schedulingapiv1.SchemeGroupVersion,
		flowcontrolv1beta1.SchemeGroupVersion,
	)
	// enable non-deprecated beta resources in extensions/v1beta1 explicitly so we have a full list of what's possible to serve
	ret.EnableResources(
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("ingresses"),
	)
	// disable alpha versions explicitly so we have a full list of what's possible to serve
	ret.DisableVersions(
		apiserverinternalv1alpha1.SchemeGroupVersion,
		nodev1alpha1.SchemeGroupVersion,
		rbacv1alpha1.SchemeGroupVersion,
		schedulingv1alpha1.SchemeGroupVersion,
		storageapiv1alpha1.SchemeGroupVersion,
		flowcontrolv1alpha1.SchemeGroupVersion,
	)

	return ret
}

```

### GVR存储时转换

StorageFactoryRestOptionsFactory的核心是StorageFactory，StorageFactory接口由DefaultStorageFactory对象实现。而DefaultStorageFactory对象则在buildGenericConfig的completedStorageFactoryConfig.New()被创建。
```Golang
// StorageFactory is the interface to locate the storage for a given GroupResource
type StorageFactory interface {
	// New finds the storage destination for the given group and resource. It will
	// return an error if the group has no storage destination configured.
	NewConfig(groupResource schema.GroupResource) (*storagebackend.Config, error)

	// ResourcePrefix returns the overridden resource prefix for the GroupResource
	// This allows for cohabitation of resources with different native types and provides
	// centralized control over the shape of etcd directories
	ResourcePrefix(groupResource schema.GroupResource) string

        // ETCD的server URL和TLS配置
	// Backends gets all backends for all registered storage destinations.
	// Used for getting all instances for health validations.
	Backends() []Backend
}
```

Factory只有一个，但每种GroupResource都重新copy一个新的配置
```
// 存储到ETCD中的各种参数，包括但不限于是否所有key的统一前缀、是否分页、etcd server的连接参数、编解码、Compaction时间间隔、健康检查、租约等
// New finds the storage destination for the given group and resource. It will
// return an error if the group has no storage destination configured.
func (s *DefaultStorageFactory) NewConfig(groupResource schema.GroupResource) (*storagebackend.Config, error) {
        // 按completedStorageFactoryConfig.New()定义的顺序，处理同一种Resource真正选择存储哪个Group到ETCD，比如apps.deployments和extensions.deployments
	chosenStorageResource := s.getStorageGroupResource(groupResource)


	// operate on copy。
	storageConfig := s.StorageConfig
	codecConfig := StorageCodecConfig{
		StorageMediaType:  s.DefaultMediaType,
		StorageSerializer: s.DefaultSerializer,
	}

        // 精确的GroupResource的ETCD配置选项会覆盖模糊的Group(Resource为"*")
	if override, ok := s.Overrides[getAllResourcesAlias(chosenStorageResource)]; ok {
		override.Apply(&storageConfig, &codecConfig)
	}
	if override, ok := s.Overrides[chosenStorageResource]; ok {
		override.Apply(&storageConfig, &codecConfig)
	}

	// 确定ETCD中的版本
	codecConfig.StorageVersion, err = s.ResourceEncodingConfig.StorageEncodingFor(chosenStorageResource)
	if err != nil {
		return nil, err
	}
	// 确定内存中的版本
	codecConfig.MemoryVersion, err = s.ResourceEncodingConfig.InMemoryEncodingFor(groupResource)
	if err != nil {
		return nil, err
	}
	codecConfig.Config = storageConfig

	storageConfig.Codec, storageConfig.EncodeVersioner, err = s.newStorageCodecFn(codecConfig)
	if err != nil {
		return nil, err
	}
	klog.V(3).Infof("storing %v in %v, reading as %v from %#v", groupResource, codecConfig.StorageVersion, codecConfig.MemoryVersion, codecConfig.Config)

	return &storageConfig, nil
}

```

返回该GroupResource的存储在ETCD中的特定前缀。选择的优先顺序是该GroupResource精确的前缀 > 该Group的前缀（Resource为"*"）> Resource的小写
```Golang
func (s *DefaultStorageFactory) ResourcePrefix(groupResource schema.GroupResource) string {
	chosenStorageResource := s.getStorageGroupResource(groupResource)
	groupOverride := s.Overrides[getAllResourcesAlias(chosenStorageResource)]
	exactResourceOverride := s.Overrides[chosenStorageResource]

	etcdResourcePrefix := s.DefaultResourcePrefixes[chosenStorageResource]
	if len(groupOverride.etcdResourcePrefix) > 0 {
		etcdResourcePrefix = groupOverride.etcdResourcePrefix
	}
	if len(exactResourceOverride.etcdResourcePrefix) > 0 {
		etcdResourcePrefix = exactResourceOverride.etcdResourcePrefix
	}
	if len(etcdResourcePrefix) == 0 {
		etcdResourcePrefix = strings.ToLower(chosenStorageResource.Resource)
	}

	return etcdResourcePrefix
}
```

## 构建http Server

回到CreateServerChain，CreateKubeAPIServer函数根据之前的Config创建kubeAPIServer实例对象，
```Golang
// CreateKubeAPIServer creates and wires a workable kube-apiserver
func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	// XXX
	return kubeAPIServer, nil
}

// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
//   KubeletClientConfig
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	// XXX
	// 创建GenericAPIServer对象，可以传入代理对象
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
	// XXX
	m := &Instance{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

        // 安装API
	// XXX
	
	// 添加GenericAPIServer hook
	// XXX
	
	return m, nil
}
```

NewAPIServerHandler将delegationTarget的handler和当前的handler穿成一个链
```Golang
// New creates a new server which logically combines the handling chain with the passed server.
// name is used to differentiate for logging. The handler chain in particular can be difficult as it starts delegating.
// delegationTarget may not be nil.
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
	// XXX
	handlerChainBuilder := func(handler http.Handler) http.Handler {
	        // BuildHandlerChainFunc就是前面<<http Server响应流程>>一节中的http handler链：DefaultBuildHandlerChain，
		return c.BuildHandlerChainFunc(handler, c.Config)
	}
	// 传入delegationTarget的handler
	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
	
	s := &GenericAPIServer{
	        // XXX
		delegationTarget:           delegationTarget,
		Handler: apiServerHandler,
		listedPathProvider: apiServerHandler,
	}
	// 安装API
	// XXX

	// use the UnprotectedHandler from the delegation target to ensure that we don't attempt to double authenticator, authorize,
	// or some other part of the filter chain in delegation cases.
	if delegationTarget.UnprotectedHandler() == nil && c.EnableIndex {
	        // 正常的NotFound由delegationTarget处理，如果它的handler为空，则设置NotFound的handler返回404
		s.Handler.NonGoRestfulMux.NotFoundHandler(routes.IndexLister{
			StatusCode:   http.StatusNotFound,
			PathProvider: s.listedPathProvider,
		})
	}

	return s, nil
}
```

APIServerHandler由三部分组成，其中GoRestfulContainer真实是一个开源库的WebService，Director作为handler代理组合了GoRestfulContainer和NonGoRestfulMux的内部处理细节。
```
// FullHandlerChain -> Director -> {GoRestfulContainer,NonGoRestfulMux} based on inspection of registered web services
type APIServerHandler struct {
	// FullHandlerChain is the one that is eventually served with.  It should include the full filter
	// chain and then call the Director.
	FullHandlerChain http.Handler
	// The registered APIs.  InstallAPIs uses this.  Other servers probably shouldn't access this directly.
	GoRestfulContainer *restful.Container
	// NonGoRestfulMux is the final HTTP handler in the chain.
	// It comes after all filters and the API handling
	// This is where other servers can attach handler to various parts of the chain.
	NonGoRestfulMux *mux.PathRecorderMux

	// Director is here so that we can properly handle fall through and proxy cases.
	// This looks a bit bonkers, but here's what's happening.  We need to have /apis handling registered in gorestful in order to have
	// swagger generated for compatibility.  Doing that with `/apis` as a webservice, means that it forcibly 404s (no defaulting allowed)
	// all requests which are not /apis or /apis/.  We need those calls to fall through behind goresful for proper delegation.  Trying to
	// register for a pattern which includes everything behind it doesn't work because gorestful negotiates for verbs and content encoding
	// and all those things go crazy when gorestful really just needs to pass through.  In addition, openapi enforces unique verb constraints
	// which we don't fit into and it still muddies up swagger.  Trying to switch the webservices into a route doesn't work because the
	//  containing webservice faces all the same problems listed above.
	// This leads to the crazy thing done here.  Our mux does what we need, so we'll place it in front of gorestful.  It will introspect to
	// decide if the route is likely to be handled by goresful and route there if needed.  Otherwise, it goes to PostGoRestful mux in
	// order to handle "normal" paths and delegation. Hopefully no API consumers will ever have to deal with this level of detail.  I think
	// we should consider completely removing gorestful.
	// Other servers should only use this opaquely to delegate to an API server.
	Director http.Handler
}
```

传入的notFoundHandler是delegationTarget.UnprotectedHandler()，delegationTarget也是一个GenericAPIServer对象，UnprotectedHandler函数内部也是delegationTarget的Director，最终形成完整的Director handler链
```Golang
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
	}

	gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(s, panicReason, httpWriter)
	})
	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
		serviceErrorHandler(s, serviceErr, request, response)
	})

	director := director{
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}

	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
}

func (s *GenericAPIServer) UnprotectedHandler() http.Handler {
	// when we delegate, we need the server we're delegating to choose whether or not to use gorestful
	return s.Handler.Director
}
```

APIServerHandler第一道handle是FullHandlerChain，它被Direcotr赋值；director是先过goRestfulContainer，再过nonGoRestfulMux，既当前的GenericAPIServer处理不了就委托给下一个GenericAPIServer处理
```
// ServeHTTP makes it an http.Handler
func (a *APIServerHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	a.FullHandlerChain.ServeHTTP(w, r)
}

func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	path := req.URL.Path

	// check to see if our webservices want to claim this path
	for _, ws := range d.goRestfulContainer.RegisteredWebServices() {
		switch {
		case ws.RootPath() == "/apis":
			// if we are exactly /apis or /apis/, then we need special handling in loop.
			// normally these are passed to the nonGoRestfulMux, but if discovery is enabled, it will go directly.
			// We can't rely on a prefix match since /apis matches everything (see the big comment on Director above)
			if path == "/apis" || path == "/apis/" {
				klog.V(5).Infof("%v: %v %q satisfied by gorestful with webservice %v", d.name, req.Method, path, ws.RootPath())
				// don't use servemux here because gorestful servemuxes get messed up when removing webservices
				// TODO fix gorestful, remove TPRs, or stop using gorestful
				d.goRestfulContainer.Dispatch(w, req)
				return
			}

		case strings.HasPrefix(path, ws.RootPath()):
			// ensure an exact match or a path boundary match
			if len(path) == len(ws.RootPath()) || path[len(ws.RootPath())] == '/' {
				klog.V(5).Infof("%v: %v %q satisfied by gorestful with webservice %v", d.name, req.Method, path, ws.RootPath())
				// don't use servemux here because gorestful servemuxes get messed up when removing webservices
				// TODO fix gorestful, remove TPRs, or stop using gorestful
				d.goRestfulContainer.Dispatch(w, req)
				return
			}
		}
	}

	// if we didn't find a match, then we just skip gorestful altogether
	klog.V(5).Infof("%v: %v %q satisfied by nonGoRestful", d.name, req.Method, path)
	d.nonGoRestfulMux.ServeHTTP(w, req)
}
```

## 安装API Routes
**k8s的API分三种**：
1. core API（又称legacy API）:路径是/api/v1
2. named group API:路径是/apis/${GROUP}/${VERSION}
3. discovery API(待验证):针对1和2，路径分别是/api/${GROUP} 和/apis/${GROUP} 
4. others API:路径是/metrics,/healthz,/version,/(根路径),/debug等
</br>

**Resource的路径命名规则是**：
1. 无namespace: 如/apis/${GROUP}/${VERSION}/resource/${NAME}
2. 有namespace: 如/apis/${GROUP}/${VERSION}/namespaces/${NAMESPACE}/resource/${NAME}
</br>

再次回到kubeAPIServerConfig.Complete().New(delegateAPIServer)。安装资源API Route分legacy和普通group两个阶段。
```Golang
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	// 创建GenericAPIServer对象，构建Handler链
	// XXX
	
	// 下面介绍
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	// XXX
	
	// install legacy rest storage
	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ExtendExpiration:            c.ExtraConfig.ExtendExpiration,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
		}
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}

	// The order here is preserved in discovery.
	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
	// with specific priorities.
	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
	// handlers that we have.
	restStorageProviders := []RESTStorageProvider{
		apiserverinternalrest.StorageProvider{},
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
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.StorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}

	// 添加GenericAPIServer hook
	// XXX
	
	return m, nil
}

// c.GenericConfig.New("kube-apiserver", delegationTarget)
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
	// 构建handler链
	// XXX
	
	s.listedPathProvider = routes.ListedPathProviders{s.listedPathProvider, delegationTarget}

        // 安装others API
	installAPI(s, c.Config)

        // 设置handler链的默认NotFound处理函数
	// XXX

	return s, nil
}
```

先说安装legacy API
```Golang
// InstallLegacyAPI will install the legacy APIs for the restStorageProviders if they are enabled.
func (m *Instance) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) error {
        // 返回的LegacyRESTStorage保存用RangeAllocation对象描述的集群全局的Range分配信息，比如CIDR范围，主机端口号等
	// legacy API的group内部用空串""表示，apiGroupInfo保存了关于legacy API全部的version->resource->storage关系，storage封装了各种resource的增删查改细节
	legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
	// XX

        // 创建负责维护集群系统级namespace和service的controller
	controllerName := "bootstrap-controller"
	coreClient := corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
	bootstrapController := c.NewBootstrapController(legacyRESTStorage, coreClient, coreClient, coreClient, coreClient.RESTClient())
	m.GenericAPIServer.AddPostStartHookOrDie(controllerName, bootstrapController.PostStartHook)
	m.GenericAPIServer.AddPreShutdownHookOrDie(controllerName, bootstrapController.PreShutdownHook)

        // DefaultLegacyAPIPrefix是/api
	if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}

func (s *GenericAPIServer) InstallLegacyAPIGroup(apiPrefix string, apiGroupInfo *APIGroupInfo) error {
	// XXX
	
	if err := s.installAPIResources(apiPrefix, apiGroupInfo, openAPIModels); err != nil {
		return err
	}

        // 安装legacy API的discovery API之一，如/api，返回的是/api下的所有版本
	// Install the version handler.
	// Add a handler at /<apiPrefix> to enumerate the supported api versions.
	s.Handler.GoRestfulContainer.Add(discovery.NewLegacyRootAPIHandler(s.discoveryAddresses, s.Serializer, apiPrefix).WebService())

	return nil
}

// installAPIResources is a private method for installing the REST storage backing each api groupversionresource
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, openAPIModels openapiproto.Models) error {
	var resourceInfos []*storageversion.ResourceInfo
	// 遍历该group下的每个version，分别注册各种资源的api
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
		if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
			klog.Warningf("Skipping API %v because it has no resources.", groupVersion)
			continue
		}
                
		// 返回helper结构，主要保存了安装http handler过程中要用到的所有信息，包括restStorage
		apiGroupVersion := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
		// XXX
		
		r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
		if err != nil {
			return fmt.Errorf("unable to setup API %v: %v", apiGroupInfo, err)
		}
		resourceInfos = append(resourceInfos, r...)
	}
	// XXX
	return nil
}

// InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
// It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
// in a slash.
func (g *APIGroupVersion) InstallREST(container *restful.Container) ([]*storageversion.ResourceInfo, error) {
	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
	installer := &APIInstaller{
		group:             g, // 带着helper对象APIGroupVersion
		prefix:            prefix,  // 下面安装每种resource API时的webservice前缀已经是/api/${GROUP}/${VERSION}
		minRequestTimeout: g.MinRequestTimeout,
	}

	apiResources, resourceInfos, ws, registrationErrors := installer.Install()
	versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
	// 安装legacy API的另一个discovery API，如/api/v1, 返回的是/api/v1下的所有资源（legacy api只有一个v1版本）
	versionDiscoveryHandler.AddToWebService(ws)
	container.Add(ws)
	return removeNonPersistedResources(resourceInfos), utilerrors.NewAggregate(registrationErrors)
}

// Install handlers for API resources.
func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
	// XXX
	// 用上面installer传入的prefix初始化，所以下面安装的都是resource API
	ws := a.newWebService()

	// Register the paths in a deterministic (sorted) order to get a deterministic swagger spec.
	paths := make([]string, len(a.group.Storage))
	var i int = 0
	for path := range a.group.Storage {
		paths[i] = path
		i++
	}
	sort.Strings(paths)
	for _, path := range paths {
	        // 遍历每周resource，path就是Resource
		apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
		// XXX
	}
	return apiResources, resourceInfos, ws, errors
}

func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
        // XX
	
	// resource有可能带着subresource
	resource, subresource, err := splitSubresource(path)
	// XXX
	
        // 获取resource对应的GVK,不太明白下面多个类似的操作怎么做的？
	fqKindToRegister, err := GetResourceKind(a.group.GroupVersion, storage, a.group.Typer)
	// XX
	versionedPtr, err := a.group.Creater.New(fqKindToRegister)
	// XX
	defaultVersionedObject := indirectArbitraryPointer(versionedPtr)
	//XX
	
	// 判断有对象支持哪些增删查改操作
	// what verbs are supported by the storage, used to know what verbs we support per path
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
	gvAcceptor, _ := storage.(rest.GroupVersionAcceptor)
	// XXX
	
	// 根据resource是否是namespace的拼接API路径，流程基本相同
	// Get the list of actions for the given scope.
	switch {
	case !namespaceScoped:
		// 非namespace的路径是/${PREFIX}/{GROUP}/${VERSION}/resource/${RESOURCE}
		// 选择添加action的Verb
		// Handler for standard REST verbs (GET, PUT, POST and DELETE).
		// Add actions at the resource path: /api/apiVersion/resource
		actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
		actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
		actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
		// DEPRECATED in 1.11
		actions = appendIf(actions, action{"WATCHLIST", "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

		// Add actions at the item path: /api/apiVersion/resource/{name}
		actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
		if getSubpath {
			actions = appendIf(actions, action{"GET", itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
		}
		actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
		actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
		actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)
		// DEPRECATED in 1.11
		actions = appendIf(actions, action{"WATCH", "watch/" + itemPath, nameParams, namer, false}, isWatcher)
		actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
		actions = appendIf(actions, action{"CONNECT", itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)
	default:
	        // namespace的路径是/${PREFIX}/{GROUP}/${VERSION}/namespaces/${NAMESPACE}/resource/${RESOURCE}
		// XXX
	}
	
	for _, action := range actions {
		// XXX
		routes := []*restful.RouteBuilder{}
		// XXX
		switch action.Verb {
		case "GET": // Get a resource.
			var handler restful.RouteFunction
			// XX
			// getter就是storage.(rest.Getter)
			handler = restfulGetResource(getter, reqScope)
			// XX handler的各种装饰器
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				Writes(producedObject)
			// XX
			routes = append(routes, route)
		case "LIST": // List all resources of a kind.
			// XX
		case "PUT":
		// XXX
		for _, route := range routes {
			// XX
			ws.Route(route)
		}
	}
	// XX
	// Record the existence of the GVR and the corresponding GVK
	a.group.EquivalentResourceRegistry.RegisterKindFor(reqScope.Resource, reqScope.Subresource, fqKindToRegister)

	return &apiResource, resourceInfo, nil
}
```

普通group API和legacy API安装流程基本一致，除了APIGroupPrefix是/apis，以及实现了RESTStorageProvider接口的各个待安装的group之外，最终都在同一个函数installAPIResources中完成。
```Golang
// RESTStorageProvider is a factory type for REST storage.
type RESTStorageProvider interface {
	GroupName() string
	NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool, error)
}
```

NewRESTStorage是用来初始化APIGroupInfo的。APIGroupInfo包含了对每个Group可以持久化的version，以及不同resource的storage结构。
两个关键的字段如下，其中VersionedResourcesStorageMap中的storage接口就是每个resource具体的REST和StatusREST对象实现

```Golang
// Info about an API group.
type APIGroupInfo struct {
        // 代码初始化时提前注册的scheme已经有该group支持的version
	PrioritizedVersions []schema.GroupVersion
	// 是该group下，version->resource->storage关系，storage封装了各种resource的增删查改细节，跟后端存储有关
	// Info about the resources in this group. It's a map from version to resource to the storage.
	VersionedResourcesStorageMap map[string]map[string]rest.Storage
	// XXX 
}
```

以daemonset的NewRESTStorage为例，NewREST一般返回REST和StatusREST结构，初始化VersionedResourcesStorageMap对象。
```
	// daemonsets
	daemonSetStorage, daemonSetStatusStorage, err := daemonsetstore.NewREST(restOptionsGetter)
	if err != nil {
		return storage, err
	}
	storage["daemonsets"] = daemonSetStorage
	storage["daemonsets/status"] = daemonSetStatusStorage
```

REST和StatusREST内部都是Store结构。它初始化了不同resource的具体strategy，按照统一的流程处理数据读写。
```
// REST implements a RESTStorage for DaemonSets
type REST struct {
	*genericregistry.Store
	//XXX
}

// StatusREST implements the REST endpoint for changing the status of a daemonset
type StatusREST struct {
	store *genericregistry.Store
}

type Store struct {
	// XXX
	
	// CreateStrategy implements resource-specific behavior during creation.
	CreateStrategy rest.RESTCreateStrategy
	// BeginCreate is an optional hook that returns a "transaction-like"
	// commit/revert function which will be called at the end of the operation,
	// but before AfterCreate and Decorator, indicating via the argument
	// whether the operation succeeded.  If this returns an error, the function
	// is not called.  Almost nobody should use this hook.
	BeginCreate BeginCreateFunc
	// AfterCreate implements a further operation to run after a resource is
	// created and before it is decorated, optional.
	AfterCreate AfterCreateFunc

	// UpdateStrategy implements resource-specific behavior during updates.
	UpdateStrategy rest.RESTUpdateStrategy
	// BeginUpdate is an optional hook that returns a "transaction-like"
	// commit/revert function which will be called at the end of the operation,
	// but before AfterUpdate and Decorator, indicating via the argument
	// whether the operation succeeded.  If this returns an error, the function
	// is not called.  Almost nobody should use this hook.
	BeginUpdate BeginUpdateFunc
	// AfterUpdate implements a further operation to run after a resource is
	// updated and before it is decorated, optional.
	AfterUpdate AfterUpdateFunc

	// DeleteStrategy implements resource-specific behavior during deletion.
	DeleteStrategy rest.RESTDeleteStrategy
	// AfterDelete implements a further operation to run after a resource is
	// deleted and before it is decorated, optional.
	AfterDelete AfterDeleteFunc
	
	// XXX
}
```

