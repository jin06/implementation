
### Provider 定义
对应于k8s的服务的几种实现机制 

```golang
const (
	proxyModeUserspace   = "userspace"
	proxyModeIPTables    = "iptables"
	proxyModeIPVS        = "ipvs"
	proxyModeKernelspace = "kernelspace" //nolint:deadcode,varcheck
)
```
```golang 
// Provider is the interface provided by proxier implementations.
type Provider interface {
	config.EndpointSliceHandler
	config.ServiceHandler
	config.NodeHandler

	// Sync immediately synchronizes the Provider's current state to proxy rules.
	Sync()
	// SyncLoop runs periodic work.
	// This is expected to run as a goroutine or as the main loop of the app.
	// It does not return.
	SyncLoop()
}
```

### Provider 接口

- 几个handler，当enpoint service node变化时的处理函数
- sync 将新的代理规则同步到 ipvs iptables等
- syncloop 类似sync ，但是这个是死循环，不返回，一直工作 

```golang
// Provider is the interface provided by proxier implementations.
type Provider interface {
	config.EndpointSliceHandler
	config.ServiceHandler
	config.NodeHandler

	// Sync immediately synchronizes the Provider's current state to proxy rules.
	Sync()
	// SyncLoop runs periodic work.
	// This is expected to run as a goroutine or as the main loop of the app.
	// It does not return.
	SyncLoop()

}
```

config.EndpoindSliceHandler



```golang
// EndpointSliceHandler is an abstract interface of objects which receive
// notifications about endpoint slice object changes.
type EndpointSliceHandler interface {
	// OnEndpointSliceAdd is called whenever creation of new endpoint slice
	// object is observed.
	OnEndpointSliceAdd(endpointSlice *discovery.EndpointSlice)
	// OnEndpointSliceUpdate is called whenever modification of an existing
	// endpoint slice object is observed.
	OnEndpointSliceUpdate(oldEndpointSlice, newEndpointSlice *discovery.EndpointSlice)
	// OnEndpointSliceDelete is called whenever deletion of an existing
	// endpoint slice object is observed.
	OnEndpointSliceDelete(endpointSlice *discovery.EndpointSlice)
	// OnEndpointSlicesSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnEndpointSlicesSynced()
}
```

ServiceHandler

```golang
// ServiceHandler is an abstract interface of objects which receive
// notifications about service object changes.
type ServiceHandler interface {
	// OnServiceAdd is called whenever creation of new service object
	// is observed.
	OnServiceAdd(service *v1.Service)
	// OnServiceUpdate is called whenever modification of an existing
	// service object is observed.
	OnServiceUpdate(oldService, service *v1.Service)
	// OnServiceDelete is called whenever deletion of an existing service
	// object is observed.
	OnServiceDelete(service *v1.Service)
	// OnServiceSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnServiceSynced()
}
```

NodeHandler

```golang
// NodeHandler is an abstract interface of objects which receive
// notifications about node object changes.
type NodeHandler interface {
	// OnNodeAdd is called whenever creation of new node object
	// is observed.
	OnNodeAdd(node *v1.Node)
	// OnNodeUpdate is called whenever modification of an existing
	// node object is observed.
	OnNodeUpdate(oldNode, node *v1.Node)
	// OnNodeDelete is called whenever deletion of an existing node
	// object is observed.
	OnNodeDelete(node *v1.Node)
	// OnNodeSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnNodeSynced()
}
```

### Provider 实现

四种代理模式分别有各自的实现，对应的源代码在 pkg/proxy下 ，分别是 
iptables，ipvs，userspace winuserspace，winkernel . 

其中 kernel只有一个winkernel实现，先不去管它。userspace是将 kube-proxy本身作为了代理，请求先发到了kube-proxy，kube-proxy自己再和对应的pod建立连接，转发请求。
iptables和ipvs是类似的，都是利用了linux的Netfilter机制。 没有本质区别，只有性能区别。






