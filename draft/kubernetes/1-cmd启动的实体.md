
#### cmd 目录结构

cmd 中是k8s可以独立运行的实例（进程）。k8s使用的是cobra封装入口和命令行等启动相关的部分。
里边有这么多子目录。

    OWNERS                   
    genswaggertypedocs       
    // 集群部署工具。节点管理
    kubeadm
    clicheck                 
    genutils                 
    kubectl  // client 工具
    // 云的控制平面组件。云控制器连接到云厂商的api
    cloud-controller-manager 
    genyaml                  
    kubectl-convert
    dependencycheck          
    importverifier           
    // 每个node上运行一个。监听资源定义变化，使得pod满足期望的定义。
    kubelet
    dependencyverifier
    // master 节点运行，资源的增删改查。提供rest接口。controller-manager通过watch接口观察变化       
    kube-apiserver           
    kubemark
    gendocs                  
    // 监控apiserver，监控资源变化，调整整个集群处于预期状态。由一系列的控制器组成
    kube-controller-manager  
    linkcheck
    genkubedocs              
    // 负责service 负载均衡。每个节点都会运行一个。负责pod的网络代理。现在的k8s版本中主要是监听service和endpoints的变化，更新本地的ipvs或者是iptables规则
    kube-proxy               
    preferredimports
    genman    
    // pod 调度器。将pod调度到合适的node上。运行在master节点上               
    kube-scheduler