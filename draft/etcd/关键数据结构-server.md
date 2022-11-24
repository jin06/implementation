

### Etcd

Etcd

server/embed/etcd.go:66

```golang
// Etcd contains a running etcd server and its listeners.
type Etcd struct {
	Peers   []*peerListener
	Clients []net.Listener
	// a map of contexts for the servers that serves client requests.
	sctxs            map[string]*serveCtx
	metricsListeners []net.Listener

	tracingExporterShutdown func()

	Server *etcdserver.EtcdServer

	cfg   Config
	stopc chan struct{}
	errc  chan error

	closeOnce sync.Once
}
```

### Etcd.Peers

server/embed/etcd.go:84

```golang
type peerListener struct {
	net.Listener
	serve func() error
	close func(context.Context) error
}
```

### Etcd.Server

server/etcdserver/server.go:208

```golang
// EtcdServer is the production implementation of the Server interface
type EtcdServer struct {
	r            raftNode                 // uses 64-bit 
	Cfg     config.ServerConfig
}

```

### Etcd -> Server -> raftNode -> raftNodeConfig -> Node 

Node是一个接口，具体实现在 

raft/node.go:255

```golang
type node struct {
	propc      chan msgWithResult
	recvc      chan pb.Message
	confc      chan pb.ConfChangeV2
	confstatec chan pb.ConfState
	readyc     chan Ready
	advancec   chan struct{}
	tickc      chan struct{}
	done       chan struct{}
	stop       chan struct{}
	status     chan chan Status

	rn *RawNode
}

```

