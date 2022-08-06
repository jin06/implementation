# Prometheus Server的启动

Prometheus 有两个程序入口，其中一个是Prometheus的维护工具，而Server的入口在 cmd/prometheus/main.go 。
<br>
初始化函数 init()做了两件事情，第一个使用自己的clientSDK注册了一个Version的指标。初始化了一个默认的数据保留时间 15d。
没看到这两个操作的必要性，不用管它。
<br>
跳到main()函数处, 第一行就是如果是debug模式下，设置一下Profile采样率，20个样本采样一次。我们不关心它的debug模式，不用深究。
    
    if os.Getenv("DEBUG") != "" {
		runtime.SetBlockProfileRate(20)
		runtime.SetMutexProfileFraction(20)
	}
然后就是一大坨代码，初始化配置，从配置文件和命令行解析出各种参数，对参数进行一些处理。
<br>
Prometheus 代码启动时有两种模式，分别是agent和server模式。代码写的比较凌乱，agent和server是混合在一起, 大量用if else 判断自身处于什么模式，做一些相应的操作。中间穿插了各种对参数的检查。
创建了很多manager（根据不同模式创建的不同），主要的有：

- webHandler web 服务相关
- 存储相关，包括了本地，远程
- 抓取相关的，scrapeManager
- 服务发现的 discoveryManagerScrape
- 通知模块 notifierManager
- 告警的
- 还有很多其他的

然后启动它们。main函数就结束了. 启动这些模块的时候，使用了这个代码库 github.com/oklog/run"。 
使用其中的Group这个结构创建一个对象来管理上面那些模块的启动和退出后的处理, 也就是管理这些组件如何启动，退出后如何处理的问题。
如果有一个组件退出了，那么所有的组件都会调用退出的逻辑，整个程序也会最后退出。这段是Group的结构，其中哪个actors就是prometheus的组件，
Group run起来以后会监听从组件中发回的错误，然后调用组件注册的退出时需要执行的逻辑。


    func (g *Group) Run() error {
	if len(g.actors) == 0 {
		return nil
	}

	// Run each actor.
	errors := make(chan error, len(g.actors))
	for _, a := range g.actors {
		go func(a actor) {
			errors <- a.execute()
		}(a)
	}

	// Wait for the first actor to stop.
	err := <-errors

	// Signal all actors to stop.
	for _, a := range g.actors {
		a.interrupt(err)
	}

	// Wait for all actors to stop.
	for i := 1; i < cap(errors); i++ {
		<-errors
	}

	// Return the original error.
	return err
}