

> 函数入口地址 pkg/scheduler/schedule_one.go:67 

> 函数定义 func (sched *Scheduler) scheduleOne(ctx context.Context) {

> ... 

> }


### 调度过程
1. 从调度队列中取一个待分配的pod信息。从队列中取的是一个 framework.QueuedPodInfo 结构.该结构中存有v1.Pod 信息
2. 调用方法frameworkForPod，根据pod中spec的schedulerName获取该pod的framework信息
    -  framework 是一个接口，集合了调度pod的插件。
3. 检查是否要跳过当前的调度，方法 skipPodSchedule
    - 如果pod已经被删除则跳过调度，主要通过判断pod的deletionTimestamp
    - 如果pod再次被修改过则跳过（修改过的pod会再次进入到调度队列）
4. 调用SchedulePod方法，传入参数一个context，framework，state，pod。 其中state是做一些统计工作。计算出适合该pod的node，将计算的结果返回。具体实现调用了 pkg/scheduler/schedule_one.go:312
    - 寻找适合该pod的node，如果没有则返回。如果只有一个node则返回该node。如果有多个node则进入下一步
    - 如果存在多个node可以选择，则计算这些node的分数
        - 如果没有framework插件，则都返回1分
        - 否则根据framework定义的计算分数的方法计算分数。
        - 再计算调度程序本身的扩展，计算分数。最后将两个分数加权后计算出最终的分数列表返回
        - 返回最高分的node
5. 计算出的最佳node绑定到相关的pod中，这里是预绑定，并非最终调度到该node
6. 执行framework的reserve 和 permit插件
7. 绑定pod。这一步起了一个goroutine异步执行
    - 他这部分写的比较绕，也有很多异常处理，出错后怎么回滚。这部分最终会调用绑定的函数，默认的绑定方式就是调用k8s的client，生成binding数据写入到k8s api接口中。
    - 绑定的函数是可以有多个的。也就是说除了调用k8s的client，也可以增加其他的bind方法，顺序执行完就结束了。

