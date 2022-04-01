# 简介
三种内置ApiServer分别是:
1. 标准的kubeApiServer：服务的对象是固定的核心resource，路径是/api和/apis
2. 扩展的APIExtensionsServer: 服务的对象是CRD，路径是/apis/extensions。目的是让用户扩展自己的Kind，但还使用K8S原生的api
3. 聚合的AggregatorServer：用户通过注册ApiService对象，可以在根路径/apis/下，定义自己的API路径，appregatorServer会将请求转发到用户扩展server上
</br>
如下是他们之间的关系和差异

# 关键实现

Restful请求的handler是以delegate模式建立的，从CreateServerChain函数可看到，请求处理的顺序是AggregatorServer->KubeAPIServer->APIExtensionsServer，而且APIExtensionsServer和AggregatorServer的config都是以一个kubeAPIServerConfig.GenericConfig对象为基础模板，各自修改相关的参数形成自己的Config
```Golang
// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
	kubeAPIServerConfig, XXX = CreateKubeAPIServerConfig(completedOptions)
	// XXX

	// If additional API servers are added, they should be gated.
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, //XXX)
	// XXX
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
	// XXX

	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
	// XXX

	// aggregator comes last in the chain
	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, //XXX)
	// XXX
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer,  //XXX)
	// XXX

	return aggregatorServer, nil
}
```


```Golang
func createAPIExtensionsConfig(
	kubeAPIServerConfig genericapiserver.Config,
	externalInformers kubeexternalinformers.SharedInformerFactory,
	pluginInitializers []admission.PluginInitializer,
	commandOptions *options.ServerRunOptions,
	masterCount int,
	serviceResolver webhook.ServiceResolver,
	authResolverWrapper webhook.AuthenticationInfoResolverWrapper,
) (*apiextensionsapiserver.Config, error) {
	// make a shallow copy to let us twiddle a few things
	// most of the config actually remains the same.  We only need to mess with a couple items related to the particulars of the apiextensions
	// 拷贝基础模板对象，在拷贝上修改参数
	genericConfig := kubeAPIServerConfig
	genericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
	genericConfig.RESTOptionsGetter = nil

	// override genericConfig.AdmissionControl with apiextensions' scheme,
	// because apiextensions apiserver should use its own scheme to convert resources.
	// TODO 没看明白apiextensions的scheme怎么起作用的？
	err := commandOptions.Admission.ApplyTo(
		&genericConfig,
		externalInformers,
		genericConfig.LoopbackClientConfig,
		feature.DefaultFeatureGate,
		pluginInitializers...)
	// XXX

	// copy the etcd options so we don't mutate originals.
	etcdOptions := *commandOptions.Etcd
	etcdOptions.StorageConfig.Paging = utilfeature.DefaultFeatureGate.Enabled(features.APIListChunking)
	// this is where the true decodable levels come from. // apiextendions的scheme，group是apiextensions.k8s.io
	etcdOptions.StorageConfig.Codec = apiextensionsapiserver.Codecs.LegacyCodec(v1beta1.SchemeGroupVersion, v1.SchemeGroupVersion)
	// prefer the more compact serialization (v1beta1) for storage until http://issue.k8s.io/82292 is resolved for objects whose v1 serialization is too big but whose v1beta1 serialization can be stored
	etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1beta1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
	genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions}

	// override MergedResourceConfig with apiextensions defaults and registry
	if err := commandOptions.APIEnablement.ApplyTo(
		&genericConfig,
		apiextensionsapiserver.DefaultAPIResourceConfigSource(),
		apiextensionsapiserver.Scheme); err != nil {
		return nil, err
	}

	apiextensionsConfig := &apiextensionsapiserver.Config{
		GenericConfig: &genericapiserver.RecommendedConfig{
			Config:                genericConfig,
			SharedInformerFactory: externalInformers,
		},
		ExtraConfig: apiextensionsapiserver.ExtraConfig{
			CRDRESTOptionsGetter: apiextensionsoptions.NewCRDRESTOptionsGetter(etcdOptions),
			MasterCount:          masterCount,
			AuthResolverWrapper:  authResolverWrapper,
			ServiceResolver:      serviceResolver,
		},
	}

	// we need to clear the poststarthooks so we don't add them multiple times to all the servers (that fails)
	apiextensionsConfig.GenericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}

	return apiextensionsConfig, nil
}
```
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
