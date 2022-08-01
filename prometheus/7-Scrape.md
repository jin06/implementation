
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

new 函数

```golang
func newScrapeLoop(ctx context.Context,...) ... {
    ...
    ...
    // buffers 在这里是一个pool
    // 这个池子里放的是不同大小的slice。这就够了。
    if buffers == nil {
		buffers = pool.New(1e3, 1e6, 3, func(sz int) interface{} { return make([]byte, 0, sz) })
	}
    // 创建一个cache
    if cache == nil {
		cache = newScrapeCache()
	}
    ...
}

```


run 函数

```golang

// 
func (sl *scrapeLoop) run(errc chan<- error) {
    ...
    // 每隔一段时间（采集的间隔时间）调用scrapeAndReport，所以这里就是开启循环，一些对循环的控制。主要采集逻辑在函数中。
    for {
        ...
        		last = sl.scrapeAndReport(last, scrapeTime, errc)
                select {
                    ...
                    ...
                    case  <-ticker.C:
                }
        ...
    }
    ...
}
```

scrapeAndReport 方法

这个方法看名字就是两部分功能。 scrape report
按照它自己的注释说明，他的功能是执行scrape并把结果追加到storage中，并且报告监控指标？



```golang
func (sl *scrapeLoop) scrapeAndReport(last, appendTime time.Time, errc chan<- error) time.Time {
    ...
    // 这里利用它自己实现的一个pool获取一个切片，切片的空间大小是上一次拉取的大小。为了减少向操作系统申请内存的次数，但是这个大小是上一次的大小感觉是太直观，不知道prometheus有没有验证过是不是最优化的方案。
    // 这个pool底层的逻辑使用的是sync包中的对象池，然后进行简单封装了一下
    b := sl.buffers.Get(sl.lastScrapeSize).([]byte)

    ...
    // app 是 storage.Appender 
    app := sl.appender(sl.appenderCtx)
    // 最后这个app提交数据，如果出错就rollback
    defer func() {
        if err!=nil {
            app.Rollback()
            return
        }
        err = app.Commit()
        ...
    }
    defer func() {
        // 调用report
		if err = sl.report(app, appendTime, time.Since(start), total, added, seriesAdded, bytes, scrapeErr); err != nil {
			level.Warn(sl.l).Log("msg", "Appending scrape report failed", "err", err)
		}
    }()
    。。。
    // 这个是请求http拿到节点的监控数据
    	contentType, scrapeErr = sl.scraper.scrape(scrapeCtx, buf)
    。。。
    // 做append操作,如果失败了，就再次写入一次（写入的内容是空的），如果还是失败那么就返回错误，这一段感觉是很随意的再修复什么bug的代码
    total, added, seriesAdded, appErr = sl.append(app, b, contentType, appendTime)
    ...

}
```

上面的调用append方法如下： scrape/scrape.go:1446

```golang
func (sl *scrapeLoop) append(app storage.Appender, b []byte, contentType string, ts time.Time) (total, added, seriesAdded int, err error) {
    // 返回一个Parser，这个是将字节数组转化成prometheus的model，这是一个迭代器
    p, err := textparse.New(b, contentType)
    ...
    for {
		var (
			et          textparse.Entry
			sampleAdded bool
		)
		if et, err = p.Next(); err != nil {
			if err == io.EOF {
				err = nil
			}
			break
		}
        // 解析的数据写入缓存
		switch et {
		case textparse.EntryType:
			sl.cache.setType(p.Type())
			continue
		case textparse.EntryHelp:
			sl.cache.setHelp(p.Help())
			continue
		case textparse.EntryUnit:
			sl.cache.setUnit(p.Unit())
			continue
		case textparse.EntryComment:
			continue
		default:
		}
		total++
        ...    
        //然后调用storage 的Append 方法 。 storage.Appender 是一个接口，这里用到的实例是manager的一个属性 append， 这里是tsdb也就是本地的db，用到的实现是 storage/fanout.go:140 fanoutAppender有一个primary属性也是Appender，最终调用的是它的方法  
		ref, err = app.Append(ref, lset, t, v)
    }
}

// primary处理的是本地的写入，secondaries是remote storage，处理远程的写入
// primary最终如果是prometheus server自带的tsdb会最终调用tsdb包中的storage的相关方法。 
// prometheus 中的存储时可以扩展的，storage更像是接口层或者说是服务层，类似操作系统的系统调用，而 tsdb更像是某一个具体的实现，处理更加具体的逻辑，比如如何组织文件，写入，载入内存等
// 具体到实现层 headAppender 的Append方法
func (f *fanoutAppender) Append(ref SeriesRef, l labels.Labels, t int64, v float64) (SeriesRef, error) {
    
	ref, err := f.primary.Append(ref, l, t, v)
	if err != nil {
		return ref, err
	}
    // 本地完成写入后，如果配置了remote 则写入远程的数据
	for _, appender := range f.secondaries {
		if _, err := appender.Append(ref, l, t, v); err != nil {
			return 0, err
		}
	}
	return ref, nil
}

//   primary 的Append方法
// 最终是增加其中两个数组的元素 samples 和 sampleSeries .增加之前有一些判断，加锁保证一致性的操作。tsdb/head_append.go:294
// 

	a.samples = append(a.samples, record.RefSample{
		Ref: s.ref,
		T:   t,
		V:   v,
	})
	a.sampleSeries = append(a.sampleSeries, s)




```

appender的commit 具体到这里，因为写入的是本地tsdb，所以具体的实现是headAppender
位置在 tsdb/head_append.go:425  
先写日志 然后处理数据保存到结构中。没有写硬盘。wal的部分可以深入在看看，这个地方好像不是实时刷硬盘，是数据到了一定的值后刷新到硬盘。




