# 客户端Informer的一般用法

下面我们以pod为例
```Golang
var ( 
    kubeClient kubernetes.Interface = xxx
    defaultResync time.Duration = 0
    informerFactory informers.SharedInformerFactory
)
informerFactory = kubeinformers.NewSharedInformerFactory(kubeClient, defaultResync)
// todo...

var (
    podInformer corev1.PodInformer
    podLister corev1listers.PodLister
    podListerSynced cache.InformerSynced
)
podInformer = ctx.informerFactory.Core().V1().Pods()
podLister = podInformer.Lister()
podListerSynced = podInformer.Informer().HasSynced

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    xxx,
		UpdateFunc: xxx,
})
if !cache.WaitForCacheSync(stopCh, c.podListerSynced) {
    return 
}
// todo...

go ctx.InformerFactory.Start(ctx.StopCh)
```

# SharedInformerFactory创建

创建sharedInformerFactory对象，注意默认是all namespaces。
```Golang
func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration) SharedInformerFactory {
	return NewSharedInformerFactoryWithOptions(client, defaultResync)
}

func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
		namespace:        v1.NamespaceAll,
		defaultResync:    defaultResync,
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory
}
```

如果不用默认option，也可以自己定义。注意关键的是WithNamespace，可以只对缓存特定的namespace对象
```Golang
type SharedInformerOption func(*sharedInformerFactory) *sharedInformerFactory

// WithCustomResyncConfig sets a custom resync period for the specified informer types.
func WithCustomResyncConfig(resyncConfig map[v1.Object]time.Duration) SharedInformerOption {
	return func(factory *sharedInformerFactory) *sharedInformerFactory {
		for k, v := range resyncConfig {
			factory.customResync[reflect.TypeOf(k)] = v
		}
		return factory
	}
}

// WithTweakListOptions sets a custom filter on all listers of the configured SharedInformerFactory.
func WithTweakListOptions(tweakListOptions internalinterfaces.TweakListOptionsFunc) SharedInformerOption {
	return func(factory *sharedInformerFactory) *sharedInformerFactory {
		factory.tweakListOptions = tweakListOptions
		return factory
	}
}

// WithNamespace limits the SharedInformerFactory to the specified namespace.
func WithNamespace(namespace string) SharedInformerOption {
	return func(factory *sharedInformerFactory) *sharedInformerFactory {
		factory.namespace = namespace
		return factory
	}
}

```

SharedInformerFactory除了是对外的接口，还有个同名内部接口。PodInformer是通过Core->V1->Pods一层一层最终创建出来的
```Golang
// SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	Admissionregistration() admissionregistration.Interface
	Apps() apps.Interface
	Auditregistration() auditregistration.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Networking() networking.Interface
	Node() node.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Scheduling() scheduling.Interface
	Settings() settings.Interface
	Storage() storage.Interface
}

// SharedInformerFactory a small interface to allow for adding an informer without an import cycle
type SharedInformerFactory interface {
	Start(stopCh <-chan struct{})
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}

func (f *sharedInformerFactory) Core() core.Interface {
	return core.New(f, f.namespace, f.tweakListOptions)
}

// New returns a new Interface.
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &group{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

// V1 returns a new v1.Interface.
func (g *group) V1() v1.Interface {
	return v1.New(g.factory, g.namespace, g.tweakListOptions)
}

func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &version{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

func (v *version) Pods() PodInformer {
	return &podInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}
}

// PodInformer provides access to a shared informer and lister for Pods.
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}
```

NewPodInformer实际调用的是NewFilteredPodInformer，使用factory共用的client建立List&Watch，以及Watch的对象类型Pod，
```Golang
// NewFilteredPodInformer constructs a new informer for Pod type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```
PodInformer
f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)

