### pod 的更新从何而来

#### 具体处理的pod的动作，处理pod的哪些变化
kubelet周期性的处理pod的变化，pod变化抽象为结构体PodUpdate，其定义为。这个结构体本质上就是将对应的pod的yaml和变化的类型表示出来。 
```golang
type PodUpdate struct {
	Pods   []*v1.Pod
	Op     PodOperation
	Source string
}
``` 
其中 Op字段定义了本次更新的类型 ，主要由一下几种，主要关注其中pod增加和删除的操作

```golang
const (
	// SET is the current pod configuration.
	SET PodOperation = iota
	// ADD signifies pods that are new to this source.
	ADD
	// DELETE signifies pods that are gracefully deleted from this source.
	DELETE
	// REMOVE signifies pods that have been removed from this source.
	REMOVE
	// UPDATE signifies pods have been updated in this source.
	UPDATE
	// RECONCILE signifies pods that have unexpected status in this source,
	// kubelet should reconcile status with this source.
	RECONCILE
)
```

#### PodUpdate 来源

PodUpdate数据会被写入到一个 chan 中，然后kubelet会监听这个chan，并周期性的从中拿到PodUpdate并进行相应的处理。

那么有哪些地方或哪些操作向这个chan中写入数据：
- 从apiServer拉取的，这部分是通过schduler分配给该节点的pod
- config中，也就是从文件中获取的，这部分不知道用来干嘛的
- http接口中，kubelet提供的api，那么可以被动的被其他人调用，实现一些手工或运维的需求

### pod启动的逻辑

定期循环后拿到了PodUpdate数据后，根据它的类型，调用相应的handler

#### 新增的逻辑
最终调用的逻辑在这里 pkg/kubelet/kubelet.go:2238， 代码很少，删除循环和无用代码后，处理每一个pod的新增的逻辑如下:

```golang
	existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		kl.podManager.AddPod(pod)

		if kubetypes.IsMirrorPod(pod) {
            // 和静态pod有关，mirror pod 是静态pod的mirror，主要方便apiserver查看
			kl.handleMirrorPod(pod, start)
			continue
		}

		// Only go through the admission process if the pod is not requested
		// for termination by another part of the kubelet. If the pod is already
		// using resources (previously admitted), the pod worker is going to be
		// shutting it down. If the pod hasn't started yet, we know that when
		// the pod worker is invoked it will also avoid setting up the pod, so
		// we simply avoid doing any work.
		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			activePods := kl.filterOutInactivePods(existingPods)

			// Check if we can admit the pod; if not, reject it.
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
```

有两个结构体比较重要，PodManager， PodWorker。 

PodManager 一个具体实现在  pkg/kubelet/pod/pod_manager.go:102。 主要做pod对象的一些管理。

PodWorker 是pod 的所欲operation的集合，也就是具体操作，按照它的注释这个对象就是在维护具体的pod对象和在运行时中的容器实现，保证运行时按照pod包含的容器定义的样子运行。具体的代码在 pkg/kubelet/pod_workers.go:379
那么pod启动的最终执行的就是这个对象的UpdatePod 方法。



### 启动容器，与容器运行时的交互

PodWorker 执行Pod的启动的方法（变更，新增、修改），不仅仅包括和运行时，也包括一些其他的逻辑一并写在这里。

代码实现的方法在 pkg/kubelet/pod_workers.go:557

通过它的注释大概了解一下这个方法的大致功能：实现pod的配置的更新的动作（也就是pod的yaml变化了，让容器也跟着变化）

这个函数代码比较多，提炼执行的主要流程：
- 如果是孤儿pod（没有对应的配置），进行一些格式转换，这里传入的参数options中有两个表示pod的字段，一个是pod另一个是runningPod，runningPod表示的就是孤儿Pod
- 决定怎么处理这个pod，启动、终止、忽略
- 如果此时要创建的pod正在执行终止操作，那么将pod sync status 的 restartRequested 字段设置为true，退出当前，然后等待以后重新创建pod。
- 如果pod已经终止，那么就退出，不能再使用相同的uid创建pod。
- 声明变量 becamingTerminating bool类型。
    - 如果是孤儿pod，设置该pod状态删除
    - 如果 DeletionTimestamp 不为空，标记为删除
    - 如果 pod.Status.Phase 不为空（成功或失败），设置删除
    - 如果 updateType 等于 kubetypes.SyncPodKill， 检查是否是驱逐的pod
- 如果是需要终止的pod，计算设置优雅退出的时间
- 最后需要更新的pod走向这里 pkg/kubelet/pod_workers.go:877

```golang
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
}

```

managePodLoop 的工作，不看各种情况下的处理，就主要看创建或更新pod时的操作
最主要步骤是执行syncPod
- 会涉及到是否已经存在的pod，即更新，是否是静态pod的逻辑。分支流程包括记录延迟时间等一些监控
- 网络检查，如果没有准备好网络并且只是用host network 则返回
- 创建 cgroups 详细的部分这里先不看 pkg/kubelet/cm/types.go:106
- 宿主机中创建一些存放pod数据的文件夹、挂载volumn的文件夹等
- 挂载pod需要的磁盘volumn，这部分细节也是可以单独来看的，这部分先不看 pkg/kubelet/volumemanager/volume_manager.go:106
- 获取 pullSecrets
- probManager 将pod加入进去
- 调用运行时 syncPod 

### 容器运行时的处理 

这部分单独新的markdown来记录

pkg/kubelet/container/runtime.go:98