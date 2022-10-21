
总体流程 
- 客户端通过接口watch资源，这个接口建立了连接后不断开，而是通过异步的方式将数据变更时间发送回来
- apiserver建立一个和storage的watch机制，etcd的话就是直接使用etcd的watch机制
- 


### watch api
watch api是apiserver中，用来异步通知资源变化的接口。

客户端通过informer连接这个接口，将需要监听的资源变化同步到本地，例如pod资源的监听，kubelet监听后在本地做响应的pod的启动和停止。

### 接口 

 GET /pods?watch=true 

 ### 通信过程 

 1. 建立一个websocket连接 ， 有更新就同步
 2. 不建立websocket，直接发送。

 ### ws的实现
http直接发送逻辑是一样的。就随便看一下ws的部分。

 大体逻辑位置 vendor/k8s.io/apiserver/pkg/endpoints/handlers/watch.go:284

 其中从ch中获取数据然后发送给client

 ```golang 
    ....
    ch := s.Watching.ResultChan()

	for {
		select {
		case <-done:
			return
		case event, ok := <-ch:
			if !ok {
				// End of results.
				return
			}
    .....
    			if s.UseTextFraming {
				if err := websocket.Message.Send(ws, streamBuf.String()); err != nil {
					// Client disconnect.
					return
				}
			} else {
				if err := websocket.Message.Send(ws, streamBuf.Bytes()); err != nil {
					// Client disconnect.
					return
				}
			}
    .......
 ```


 ### apiserver和etcd之间

 apiserver从持久层获取数据首先要经过Storage这个抽象层。 vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go:1252

 ```golang 
 func (e *Store) Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error) {
 }
 ```

 看一个具体的实现， etcd3的实现 staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:163

 只要的方法有两个，一个是启动watching，一个是处理取到的数据

 ```golang 
 ....
 	go wc.startWatching(watchClosedCh)
...
	go wc.processEvent(&resultChanWG)
...

 ```


其中的startWatching部分， 这部分就是调用etcd的client然后调用Watch方法监听etcd的变化，wc.key，处理一下能达到的数据

```golang
.....
	wch := wc.watcher.client.Watch(wc.ctx, wc.key, opts...)
	for wres := range wch {
		if wres.Err() != nil {
			err := wres.Err()
			// If there is an error on server (e.g. compaction), the channel will return it before closed.
			logWatchChannelErr(err)
			wc.sendError(err)
			return
		}
		if wres.IsProgressNotify() {
			wc.sendEvent(progressNotifyEvent(wres.Header.GetRevision()))
			metrics.RecordEtcdBookmark(wc.watcher.objectType)
			continue
		}

		for _, e := range wres.Events {
			parsedEvent, err := parseEvent(e)
			if err != nil {
				logWatchChannelErr(err)
				wc.sendError(err)
				return
			}
			wc.sendEvent(parsedEvent)
		}
	}
    .....
```

event的结构 
```golang 
type event struct {
	key              string
	value            []byte
	prevValue        []byte
	rev              int64
	isDeleted        bool
	isCreated        bool
	isProgressNotify bool
}
```

具体处理的逻辑，也是一个goroutine，然后循环从队列中拿event，处理event的方法如下 vendor/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:324  transform又调用了 wc.prepareObjes(e) 拿到了新旧的数据，如果是增加修改就把新的obj返回，删除就返回旧的。代码量太多了，只看最主要的流程部分。
```golang 
// transform transforms an event into a result for user if not filtered.
func (wc *watchChan) transform(e *event) (res *watch.Event) {
	curObj, oldObj, err := wc.prepareObjs(e)
	if err != nil {
		klog.Errorf("failed to prepare current and previous objects: %v", err)
		wc.sendError(err)
		return nil
	}

	switch {
	case e.isProgressNotify:
		if wc.watcher.newFunc == nil {
			return nil
		}
		object := wc.watcher.newFunc()
		if err := wc.watcher.versioner.UpdateObject(object, uint64(e.rev)); err != nil {
			klog.Errorf("failed to propagate object version: %v", err)
			return nil
		}
		res = &watch.Event{
			Type:   watch.Bookmark,
			Object: object,
		}
	case e.isDeleted:
		if !wc.filter(oldObj) {
			return nil
		}
		res = &watch.Event{
			Type:   watch.Deleted,
			Object: oldObj,
		}
	case e.isCreated:
		if !wc.filter(curObj) {
			return nil
		}
		res = &watch.Event{
			Type:   watch.Added,
			Object: curObj,
		}
	default:
		if wc.acceptAll() {
			res = &watch.Event{
				Type:   watch.Modified,
				Object: curObj,
			}
			return res
		}
		curObjPasses := wc.filter(curObj)
		oldObjPasses := wc.filter(oldObj)
		switch {
		case curObjPasses && oldObjPasses:
			res = &watch.Event{
				Type:   watch.Modified,
				Object: curObj,
			}
		case curObjPasses && !oldObjPasses:
			res = &watch.Event{
				Type:   watch.Added,
				Object: curObj,
			}
		case !curObjPasses && oldObjPasses:
			res = &watch.Event{
				Type:   watch.Deleted,
				Object: oldObj,
			}
		}
	}
	return res
}
```