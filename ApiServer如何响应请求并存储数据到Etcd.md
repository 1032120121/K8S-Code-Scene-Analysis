
ApiServer中一个资源变更请求要经历的完整流程主要包括认证(Authentication)，授权(Autoritation), 审计（Audit）, 准入（Admission）,GVK转换，对象持久化

注入插件的配置
--enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
--disable-admission-plugins=PodNodeSelector,AlwaysDeny ...

TODO：三个apiserver的关系
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
}

// 关键的配置函数
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
    // 整个函数都是在做genericConfig的配置
    genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
    genericConfig.MergedResourceConfig = controlplane.DefaultAPIResourceConfigSource()
    
    // 以下是各种ApplyTo是将命令行启动的选型配置赋值给genericConfig
    if lastErr = s.GenericServerRunOptions.ApplyTo(genericConfig); lastErr != nil {
	return
    }
    // XXX
    // Merge资源API
    if lastErr = s.APIEnablement.ApplyTo(genericConfig, controlplane.DefaultAPIResourceConfigSource(), legacyscheme.Scheme); lastErr != nil {
	return
    }
    
    
    // XXX
}
```
定义了完整的http请求处理流程
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

func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	// handler处理的顺序和此处定义的顺序一致
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
apiserver可以通过--runtime-config参数控制api/all、api/ga、api/beta、api/alpha四种内置API的启用和禁止（一般用在aggregatorServer，普通kubeApiserver不需要）。先确定了server**默认**支持的gv和gvr。再进行Merge
```Golang
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

## 全局初始化

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


http请求的handler链：DefaultBuildHandlerChain


// FullHandlerChain -> Director -> {
GoRestfulContainer,NonGoRestfulMux} based on inspection of registered web services
type APIServerHandler struct {


etcd vendor/k8s\.io/apiserver/pkg/storage/inteface.go
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
