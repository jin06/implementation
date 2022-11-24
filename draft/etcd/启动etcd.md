
### 入口

server/main.go:30 


#### 启动etcd函数

server/etcdmain/etcd.go:43

```golang
func startEtcdOrProxyV2(args []string) {
}
```

- 初始化配置，初始化日志等
	cfg := newConfig()
- 数据目录的检查
- 启动etcd 
    - server/etcdmain/etcd.go:204 
    - server/embed/etcd.go:93

server/embed/etcd.go:93

这里启动etcd和http server

```golang
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
}
```

- 生成Etcd结构体
- 配置监听集群的节点，每个其他节点都监听
- 启动EtcdServer
    - 创建Transport，节点之间发送和接收数据
- Etcd服务节点
    - 为集群中每一个节点创建一个监听
- 服务clients
- 服务监控




