```Golang
var ( 
    kubeClient kubernetes.Interface = xxx
    defaultResync time.Duration = 0
    informerFactory informers.SharedInformerFactory
)
informerFactory = kubeinformers.NewSharedInformerFactory(kubeClient, defaultResync)
```

