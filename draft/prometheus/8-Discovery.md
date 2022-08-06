### Discovery

prometheus自动发现监控节点的方式。 

就是动态的获取需要监控的节点，然后reload，重新配置抓取的scrape的。

例如k8s的注解，可以解析工作负载的注解，如果工作负载配置有相关的注解，那么prometheus就可以reload去抓取这个新的工作负载中的pod。

不重要，快速浏览一下。

main.go 中定义了两个discovery manager   cmd/prometheus/main.go:560

```golang
		discoveryManagerScrape  discoveryManager
		discoveryManagerNotify  discoveryManager
```

实例初始化的地方 discovery/manager.go:110

实现对应的结构是  discovery.Manager 
该结构有三个重要方法，本别是 
 - ApplyConfig  初始化配置，做一些初始化的工作
 - Run 运行一个goroutine， 将数据写入到syncCh中
 - SyncCh 返回第二步的chan


 上面第二步的 chan 作为参数 传入到scapeManager中， 至此，整个闭环了，scrapeManager和 discoveryManager通过这个chan 通信，当discovery发现target有更新时通知到 scrapeManager 进行reload

 多种discoveryManager 多种实现，只要实现了这三个方法就可以了


 ### 具体的实现之 kubernetes

承担具体的发现工作的部分是discovery/discovery.go:34 。 manager管理这些discoverer。 这个discoverer就只是实现了一个方法Run

这个接口传入的参数是一个chan。具体的调用代码如下 discovery/manager.go:295

```golang
func (m *Manager) startProvider(ctx context.Context, p *Provider) {
	level.Debug(m.logger).Log("msg", "Starting provider", "provider", p.name, "subs", fmt.Sprintf("%v", p.subs))
	ctx, cancel := context.WithCancel(ctx)
	updates := make(chan []*targetgroup.Group)

	p.cancel = cancel

	go p.d.Run(ctx, updates)
	go m.updater(ctx, p, updates)
}
```

这个方法不管别的就是网这个updaes这个chan中写数据。。



一个实现  kubernetes.Ingress  discovery/kubernetes/ingress.go:41

```golang
// Ingress implements discovery of Kubernetes ingress.
type Ingress struct {
	logger   log.Logger
	informer cache.SharedInformer
	store    cache.Store
	queue    *workqueue.Type
}
```

Run方法 discovery/kubernetes/ingress.go:78
```golang
// Run implements the Discoverer interface.
func (i *Ingress) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	defer i.queue.ShutDown()

	if !cache.WaitForCacheSync(ctx.Done(), i.informer.HasSynced) {
		if !errors.Is(ctx.Err(), context.Canceled) {
			level.Error(i.logger).Log("msg", "ingress informer unable to sync cache")
		}
		return
	}

	go func() {
		for i.process(ctx, ch) {
		}
	}()

	// Block until the target provider is explicitly canceled.
	<-ctx.Done()
}
```

主要逻辑在process方法 discovery/kubernetes/ingress.go:97

```golang
func (i *Ingress) process(ctx context.Context, ch chan<- []*targetgroup.Group) bool {
    //  从queue中取出一个对象。 process是处理过程，处理数据，并发送到chan中做进一步处理。
	keyObj, quit := i.queue.Get()
    ...
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
    ...
	o, exists, err := i.store.GetByKey(key)
    ...
// 根据ingress的版本处理，处理是调用的k8s的client_go 这个package
	var ia ingressAdaptor
	switch ingress := o.(type) {
	case *v1.Ingress:
		ia = newIngressAdaptorFromV1(ingress)
	case *v1beta1.Ingress:
		ia = newIngressAdaptorFromV1beta1(ingress)
 ....
 // 构建group的结构，并发到chan中，具体怎么将ingress的标签怎么处理成group就不看了。
	send(ctx, ch, i.buildIngress(ia))
	return true
}
    
}
```

如何网上面的queue中写数据？

直接调用k8s的包，他的包中有一个 SharedInformer 的结构 
    
    这个方法就是监听k8s变化的
	AddEventHandler(handler ResourceEventHandler)
