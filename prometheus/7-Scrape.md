
三个主要的部分
* Manager
* Scrape
* Target

### Manager 

主要在文件 scrape/manager.go 注释中已经说明了它是管理scrape，会实时的刷新配置，根据它从discovery manager接收的信息。
discovery应该是prometheus的scrape 发现功能。这部分代码在discovery下，里边有很多不同来源的各种各样的实现。



```golang
// Manager maintains a set of scrape pools and manages start/stop cycles
// when receiving new target groups from the discovery manager.
type Manager struct {
	opts      *Options
	logger    log.Logger
	append    storage.Appendable
	graceShut chan struct{}

	jitterSeed    uint64     // Global jitterSeed seed is used to spread scrape workload across HA setup.
	mtxScrape     sync.Mutex // Guards the fields below.
	scrapeConfigs map[string]*config.ScrapeConfig
	scrapePools   map[string]*scrapePool
	targetSets    map[string][]*targetgroup.Group

	triggerReload chan struct{}
}
```

Manager 的Run方法，这个方法也是main方法中调用的。
main中调用的代码是这样的：
> err := scrapeManager.Run(discoveryManagerScrape.SyncCh())

传入一个chan， 看Run的实现如下，先异步运行 reloader方法，然后
从chan中取数据，调用updateTsets 方法，然后异步触发reload（往manager的triggerReload 这个chan 写数据，reloader方法就是从这个chan触发的。）


```golang
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	go m.reloader()
	for {
		select {
		case ts := <-tsets:
			m.updateTsets(ts)

			select {
			case m.triggerReload <- struct{}{}:
			default:
			}

		case <-m.graceShut:
			return nil
		}
	}
}
```

reloader 触发reload的操作最终调用到了方法 scrape/manager.go:200 。

triggerReload 这个chan的缓冲只有1. 就是说同时只有一个reload方法在执行。

执行reload，遍历chan中取出的map，map的key是scrape pool的名字，val是一个group数组。

遍历这个map，通过key判断是否存在对应的scrap pool，没有则根据key初始化一个。然后调用scrp pool 的sync方法(异步)。  所有的值遍历以后，等待他们执行完成后函数结束。

scrape pool的sync方法位置在 scrape/scrape.go:474
这里先对数据进行处理，生成targets数组的数据，最终调用处理逻辑——生成新的scrape，停止旧的scrape。。

### Scrape

Scrape和Target的区别，Scrape是数据抓取，Target的是一个个终端，是等待被拉取数据的Endpoint。

Scrape 这部分包括几个结构
 * scraper targetScraper
    targetScraper 就是scraper的实现
 * scrapePool
 * scrapeLoop
 * scrapeCache

 首先他们协同工作完成了一个事情：按照程序的意图采集指定的节点的端口提供的监控数据，然后写入到文件中。。


##### targetScraper
targetScraper 的结构。有一个Target，估计就是他需要抓取的那个target，一个http client  . 这些属性也都是为了http请求服务的

 ```golang
 type targetScraper struct {
	*Target

	client  *http.Client
	req     *http.Request
	timeout time.Duration

	gzipr *gzip.Reader
	buf   *bufio.Reader

	bodySizeLimit int64
}
 ```

这个结构有一个方法是处理http请求的。处理http请求并返回结构，写入到函数调用时给的writer中。

```golang
func (s *targetScraper) scrape(ctx context.Context, w io.Writer) (string, error) {
}
```

##### scrapeCache



##### scrapeLoop 

结构如下，省略了一些。包含了一个scrper字段

```golang
type scrapeLoop struct {
	scraper         scraper
    ...
}
```

run 函数

```golang
    
```



