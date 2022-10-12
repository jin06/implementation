### SyncPod 
位置 pkg/kubelet/kuberuntime/kuberuntime_manager.go:668

该函数主要的功能就是保持容器以期望的（资源定义的）样子在节点运行。主要有以下几步
- 计算sandbox和容器的更新
- 如果需要杀死sandbox和容器
- 如果需要创建sandbox
- 如果需要创建短暂容器
- 创建初始容器
- 创建工作容器

sanbox 接口 vendor/k8s.io/cri-api/pkg/apis/services.go:68

container 接口 vendor/k8s.io/cri-api/pkg/apis/services.go:33

这两个接口的实现都是这个接口 staging/src/k8s.io/cri-api/pkg/apis/services.go:103

 这个接口的实现在 pkg/kubelet/cri/remote/remote_runtime.go:46

 这个结构体的定义是，其中runtimeClient是负责和运行时沟通的，通过grpc的方式发送命令，runtime service 相当于调用层吧

 调用的命令会走到stream结构中，stream是另外一层会适配不同的容器运行时，例如docker containerd。

 ```golang
 // remoteRuntimeService is a gRPC implementation of internalapi.RuntimeService.
type remoteRuntimeService struct {
	timeout               time.Duration
	runtimeClient         runtimeapi.RuntimeServiceClient
	runtimeClientV1alpha2 runtimeapiV1alpha2.RuntimeServiceClient
	// Cache last per-container error message to reduce log spam
	logReduction *logreduction.LogReduction
}
 ```