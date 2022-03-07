# 客户端Informer的一般用法

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

