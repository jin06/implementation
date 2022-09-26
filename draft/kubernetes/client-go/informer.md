> informer 解决的问题就是，客户端如何感知到k8s中各种资源的变化。这个是一个很基础的问题，因为k8s本身设计的核心就是以资源为中心，定义一种资源，然后各种其他的控制体监控资源的变化，根据资源定义的变化做出相应的action。  i



> informer 是kubernetes客户端api的封装。监听资源和通知资源变化的功能。

> 项目 github.com:kubernetes/client-go.git

> 目录 informers

> 查看其中一个informer


> 接口和实现，地址 informers/apps/v1/deployment.go:42

```golang 
// DeploymentInformer provides access to a shared informer and lister for
// Deployments.
type DeploymentInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.DeploymentLister
}

type deploymentInformer struct {
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}
```


### cache.SharedIndexInformer 
 
 > 具体实现 tools/cache/shared_informer.go:287
    > - 索引本地缓存 Indexer
    > - Controller: 使用ListerWatcher 拉取objects和notification并推送到队列deltaFifo。从队列中拿数据，使用 HandlerDelta处理。
    > - sharedProcessor 负责通知转发给informer的clients 

 ```golang

 ```

> ... 这部分本身有一些绕，informer方法返回的是cache包中的一个叫SharedIndexInfomer的结构

> 在生成这个informer的时候传递进去了一个watcher，这个watcher应该是调用 k8s api server后建立的一个长连接。这部分没有仔细看，但是应该能猜测到。

> informer 通过这个连接将关心的数据更新到本地，例如如果是监听pod的数据，就返回pod的各种变化的数据。

> informer 最终使用这些数据的时候，先从本地的缓存读取数据（本地缓存的数据在informer初始化后先调用k8s的api拉取全部数据，后续更新就是通过那个watcher）



`todo`

> informer 他就是这样设计的，记录未解决的问题： 那个watcher api是如何实现的，两个问题，api server 和 client-go的通信是如何实现的？api server 是怎么监听资源变化的？

> 猜测是 通信使用的是 grpc， 监控变化应该是直接使用 etcd。 