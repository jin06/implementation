### kubelete 工作

> 入口地址： cmd/kubelet/kubelet.go:34 
> main函数上方的注释已经解释了kubelet的工作内容，管理节点上的容器，具体的内容包括：从etcd server同步配置数据，和容器运行时通信，同步数据、启动或者停止容器运行。

### command 

cmd/kubelet/app/server.go:123 它的command的代码如下, Command这个包本身就是一个对go进程的启动的一个封装，方便我们写启动逻辑，处理参数和启动命令。

这个包的使用让代码可以看起来比较整洁一些，很多golang的项目都采用了这个包来封装启动流程。是根据日常设计一个进程启动命令的常见方式，让用户能够方便的实现。

看一下它自身的简介，进一步阐释了kubelet的工作内容。

- kubelet 运行在每一个工作节点上，是节点上主要的代理。（其他的还有kubeproxy是做网络的代理）
- kubelet 可以向apiserver注册这个node （使用hostname或者是其他标志，或者是一些针对云提供商的方式）
- pod管理：管理本地的容器，让这些容器按照PodSpec中定义的样子运行，还有健康检查的工作。这里讲主要是通过apiserver的方式获取PodSpec）。
还有其他的方式可以获取PodSpec的数据，这些数据就是在YAML中定义的
    - 文件 （启动的时候传入路径，这个路径下的文件每隔20s更新一次）
    - HTTP endpoint   也是20s 去服务端请求一次
    - HTTP server 自己变成了服务端，这个时候就等着别人调自己更新。为什么这么多的方式可以获取到pod呢，之前看k8s的设计，按理应该只是通过apiserver才更加合理、简洁，有点奇怪？

然后cmd本身执行的逻辑有解析参数，校验参数，权限，读配置文件，设置是否可以动态更新等等。。。这些都不重要，最后走向启动的函数。 直接去看调用的run函数

```golang 
// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()

	kubeletConfig, err := options.NewKubeletConfiguration()
	// programmer error
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}

	cmd := &cobra.Command{
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. It can register the node with the apiserver using one of: the hostname; a flag to
override the hostname; or specific logic for a cloud provider.

The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object
that describes a pod. The kubelet takes a set of PodSpecs that are provided through
various mechanisms (primarily through the apiserver) and ensures that the containers
described in those PodSpecs are running and healthy. The kubelet doesn't manage
containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are three ways that a container
manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. Files under this path will be monitored
periodically for updates. The monitoring period is 20s by default and is configurable
via a flag.

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).

HTTP server: The kubelet can also listen for HTTP and respond to a simple API
(underspec'd currently) to submit a new manifest.`,
		// The Kubelet has special flag parsing requirements to enforce flag precedence rules,
		// so we do all our parsing manually in Run, below.
		// DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
		// `args` arg to Run, without Cobra's interference.
		DisableFlagParsing: true,
		SilenceUsage:       true,
		RunE: func(cmd *cobra.Command, args []string) error {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				return fmt.Errorf("failed to parse kubelet flag: %w", err)
			}

			// check if there are non-flag arguments in the command line
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				return fmt.Errorf("unknown command %+s", cmds[0])
			}

			// short-circuit on help
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				return errors.New(`"help" flag is non-bool, programmer error, please correct`)
			}
			if help {
				return cmd.Help()
			}

			// short-circuit on verflag
			verflag.PrintAndExitIfRequested()

			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
			}

			// validate the initial KubeletFlags
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				return fmt.Errorf("failed to validate kubelet flags: %w", err)
			}

			if cleanFlagSet.Changed("pod-infra-container-image") {
				klog.InfoS("--pod-infra-container-image will not be pruned by the image garbage collector in kubelet and should also be set in the remote runtime")
			}

			// load kubelet config file, if provided
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					return fmt.Errorf("failed to load kubelet config file, error: %w, path: %s", err, configFile)
				}
				// We must enforce flag precedence by re-parsing the command line into the new object.
				// This is necessary to preserve backwards-compatibility across binary upgrades.
				// See issue #56171 for more details.
				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					return fmt.Errorf("failed to precedence kubeletConfigFlag: %w", err)
				}
				// update feature gates based on new config
				if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
				}
			}

			// Config and flags parsed, now we can initialize logging.
			logs.InitLogs()
			if err := logsapi.ValidateAndApplyAsField(&kubeletConfig.Logging, utilfeature.DefaultFeatureGate, field.NewPath("logging")); err != nil {
				return fmt.Errorf("initialize logging: %v", err)
			}
			cliflag.PrintFlags(cleanFlagSet)

			// We always validate the local configuration (command line + config file).
			// This is the default "last-known-good" config for dynamic config, and must always remain valid.
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig, utilfeature.DefaultFeatureGate); err != nil {
				return fmt.Errorf("failed to validate kubelet configuration, error: %w, path: %s", err, kubeletConfig)
			}

			if (kubeletConfig.KubeletCgroups != "" && kubeletConfig.KubeReservedCgroup != "") && (strings.Index(kubeletConfig.KubeletCgroups, kubeletConfig.KubeReservedCgroup) != 0) {
				klog.InfoS("unsupported configuration:KubeletCgroups is not within KubeReservedCgroup")
			}

			// The features.DynamicKubeletConfig is locked to false,
			// feature gate is not locked using the LockedToDefault flag
			// to make sure node authorizer can keep working with the older nodes
			if utilfeature.DefaultFeatureGate.Enabled(features.DynamicKubeletConfig) {
				return fmt.Errorf("cannot set feature gate %v to %v, feature is locked to %v", features.DynamicKubeletConfig, true, false)
			}

			// construct a KubeletServer from kubeletFlags and kubeletConfig
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}

			// use kubeletServer to construct the default KubeletDeps
			kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
			if err != nil {
				return fmt.Errorf("failed to construct kubelet dependencies: %w", err)
			}

			if err := checkPermissions(); err != nil {
				klog.ErrorS(err, "kubelet running with insufficient permissions")
			}

			// make the kubelet's config safe for logging
			config := kubeletServer.KubeletConfiguration.DeepCopy()
			for k := range config.StaticPodURLHeader {
				config.StaticPodURLHeader[k] = []string{"<masked>"}
			}
			// log the kubelet's config for inspection
			klog.V(5).InfoS("KubeletConfiguration", "configuration", config)

			// set up signal context for kubelet shutdown
			ctx := genericapiserver.SetupSignalContext()

			// run the kubelet
			return Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate)
		},
	}

	// keep cleanFlagSet separate, so Cobra doesn't pollute it with the global flags
	kubeletFlags.AddFlags(cleanFlagSet)
	options.AddKubeletConfigFlags(cleanFlagSet, kubeletConfig)
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```

### Run 函数

函数位置在cmd/kubelet/app/server.go:411，这个函数很短。看一下注释这个函数的作用，k8s的注释真的还是比较多的，直接看注释基本就能搞清楚，很多函数也不用仔细看了。但是函数命名方面不太好，风格不是很统一，还有很多的长函数。。

这个函数负责启动kubelet，根据依赖启动，这里的依赖是这个 *kubelet.Dependencies 这个结构体来表示，如果它是nil，则进行一些初始化的逻辑。

有两个主要的函数 initForOS 和 run， 从名字看一个是为了启动做的准备， 这个initForOS点进去看了下，如果是编译成windows平台的会执行相关逻辑，如果是其他的平台（linux，unix或者其他）则什么也不执行，这一步应该是设置进程的优先级，这个是进程调度的时候会使用，目的是改变占用cpu时间，这个要看具体操作系统中的实现算法，这个值其实就是其中的一个该进程的基础的变量，优先级、进程运行的时间等决定了进程调度。总之这一步也不是很重要，就是改变当前进程的优先级，那么本身kubelet是很重要的一个代理，它肯定最好是要被优先调度的。

```golang 
// Run runs the specified KubeletServer with the given Dependencies. This should never exit.
// The kubeDeps argument may be nil - if so, it is initialized from the settings on KubeletServer.
// Otherwise, the caller is assumed to have set up the Dependencies object and a default one will
// not be generated.
func Run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) error {
	// To help debugging, immediately log version
	klog.InfoS("Kubelet version", "kubeletVersion", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	if err := initForOS(s.KubeletFlags.WindowsService, s.KubeletFlags.WindowsPriorityClass); err != nil {
		return fmt.Errorf("failed OS init: %w", err)
	}
	if err := run(ctx, s, kubeDeps, featureGate); err != nil {
		return fmt.Errorf("failed to run Kubelet: %w", err)
	}
	return nil
}
```

看一下具体的run函数  cmd/kubelet/app/server.go:490  这个函数很长，有接近400行。主要逻辑：
- 使用文件锁，用来互斥启动kubelet
- 动态配置相关
- kube dependencies  这个有点乱，很乱。这个部分原本的意思是集合各种函数，处理kubelet具体的工作，比如volumn，认证等。。
- cloudprovider 如果是云原生相关的设置
- 如果是单点模式的，进行一些初始化工作（修改一些配置）
- cadvisor 初始化 ， 监控用途：容器、进程、节点、go运行时
- eventRecorder 记录事件
- containerManager  初始化
- 设置 kubelet进程的oom_adj 基础分数。操作系统内存不足时触发oom消灭进程时会计算各个进程的分数，这是一个基础的值，用户告诉自己的优先级。
- runtimeService 用来启动容器等操作的集合
- 启动kubelet  RunKubelet
- 如果设置了，启动健康检查 一个 http server

```golang
func run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) (err error) {
	// Set global feature gates based on the value on the initial KubeletServer
    // 1. 开启一些功能（都是一些新功能，不稳定的新功能）
	err = utilfeature.DefaultMutableFeatureGate.SetFromMap(s.KubeletConfiguration.FeatureGates)    
    ...
    ...
}
```

### RunKubelet

函数位置 cmd/kubelet/app/server.go:1097

执行一些初始工作，例如设置nodeName，设置kubelet进程的最大打开文件数。 最后执行 startKubelet  cmd/kubelet/app/server.go:1180

kubelet还有一个runonce的模式，即启动后执行完相关工作就退出，不是kubelet的常见方式。这部分就不管了。

然后跳入到startKubelet中查看具体流程：

```golang
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go k.Run(podCfg.Updates())

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(kubeCfg, kubeDeps.TLSOptions, kubeDeps.Auth, kubeDeps.TracerProvider)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(netutils.ParseIPSloppy(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.KubeletPodResources) {
		go k.ListenAndServePodResources()
	}
}
```

这个函数启动了一个goroutine，并根据设置启动最多3个goroutine

- kubelet Run() 启动的goroutine
	- cloudResourceManager Run
	- go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop) 
	volume manager
		- go vm.volumePluginMgr.Run(stopCh) // start informer for CSIDriver  用来更新 storage的 driver
		- go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh) 
		定时检查pod的volumn，保证他们以定义的运行。周期执行的是 pkg/kubelet/volumemanager/populator/desired_state_of_world_populator.go:172  这里的函数，执行两个findAndAddNewPods和findAndRemoveDeletedPods。 看名字就知道，一个是遍历看看新添加了的pod，然后根据执行新增加的pod的相关volumn的工作，另一个是删除的pod的相对应的volumn工作
		- go vm.reconciler.Run(stopCh) 硬盘挂载相关的管理工作
	- 同步（上报）节点状态
	go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
	go kl.fastStatusUpdateOnce()
	- 开始租期 go kl.nodeLeaseController.Run(wait.NeverStop)
	- 	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)
	注释中解释是检查容器运行时的状态
	- 同步pod状态的manager
	- 同步运行时运行时类型，容器运行时的manager
	- event相关处理
	- 循环处理pod的更新
- 根据配置开启api sever，一共三个 分别是：server 日志啊，profile什么的接口； server read only； server pod resource；
前两个是http server，最后一个是grpc的

总结一下，启动流程启动的主要东西
- volumn相关的管理器，管理不同的storage plugin，volumn的挂载等操作
- 监控相关的，容器运行时监控、节点监控、进程的监控
- 状态相关，节点状态
- pod的处理相关的
- event时间相关