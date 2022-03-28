# 场景
三种内置ApiServer分别是标准的kubeApiServer, 扩展的ExtensionsServer，以及聚合的AggregatorServer。他们之间的关系和差异是什么？

# 关键实现

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
