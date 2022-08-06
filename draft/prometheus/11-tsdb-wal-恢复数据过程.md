

之前在一家小公司出现了prometheus server频繁重启的现象（内存不足，磁盘空间不足），然后就是每次重启prometheus时，会发生一个wal的恢复过程，这个过程很漫长，分钟级别，甚至10几分钟，看一下这部分的动作以及是否有一些优化的空间。

现在已经离开了那家公司，这台出问题的prometheus的数据不是记得很准确，大概是：
* 1000 多个target
* 产生的样本数量大概是 每天 20亿条左右，那就是平均每两个小时 2亿条样本数据左右
* 每天暂用的存储空间大概是 3-5GB

### 代码快速分析

wal的结构是 tsdb/wal/wal.go:176

```golang 
type WAL struct {
	dir         string
	logger      log.Logger
	segmentSize int
	mtx         sync.RWMutex
	segment     *Segment // Active segment.
	donePages   int      // Pages written to the segment.
	page        *page    // Active page.
	stopc       chan chan struct{}
	actorc      chan func()
	closed      bool // To allow calling Close() more than once without blocking.
	compress    bool
	snappyBuf   []byte

	metrics *walMetrics
}
```

修复的方法是 tsdb/wal/wal.go:348 从这里开始, prometheus 有两个地方调用server调用的地方在 tsdb/db.go:753  
调用的代码是 , initErr 如果不为空则会调用， Repair 传递的参数正是initErr  
 ```golang
 	if initErr := db.head.Init(minValidTime); initErr != nil {
		db.head.metrics.walCorruptionsTotal.Inc()
		level.Warn(db.logger).Log("msg", "Encountered WAL read error, attempting repair", "err", initErr)
		if err := wlog.Repair(initErr); err != nil {
			return nil, errors.Wrap(err, "repair corrupted WAL")
		}
	}
 ```

上面提到调用repair的地方和条件，具体看一下initErr是否为空的条件
调用的地方是db.head.Init db的head初始化部分。那么wal的replaying部分在这里，repair并不是耗更多时间的地方
tsdb/head.go:479


```golang
    ... 
    // 前面做了一些操作省略掉，然后开始迭代恢复内存中的数据
    // 加载数据，并没有做过多的逻辑转换，从prometheus日志也能看到这部分不是耗时最关键的部分
	for i := startFrom; i <= endAt; i++ {
		s, err := wal.OpenReadSegment(wal.SegmentName(h.wal.Dir(), i))
        。。。
		sr, err := wal.NewSegmentBufReaderWithOffset(offset, s)
        。。。
   		
		err = h.loadWAL(wal.NewReader(sr), multiRef, mmappedChunks)
		if err := sr.Close(); err != nil {
			level.Warn(h.logger).Log("msg", "Error while closing the wal segments reader", "err", err)
		}                  
    }
        ...

```

tsdb/head_wal.go:44 查看这个loadWal方法

调用的代码是，看起来像是一块堆空间。 加载部分用到了mmap 

	mmappedChunks, err := h.loadMmappedChunks(refSeries)


接着往下 这里应该是耗时最多的地方
```golang
 	for i := startFrom; i <= endAt; i++ {
         // 打开一个segment文件，构建一个Segment结构，里边包含了 fd，返回的就是这个Segment，这里没做什么其他操作
		s, err := wal.OpenReadSegment(wal.SegmentName(h.wal.Dir(), i))
...

		offset := 0
...
// 这里只是建立一个缓冲，提升io性能，没有什么问题
		sr, err := wal.NewSegmentBufReaderWithOffset(offset, s)
        ...
        // NewReader 返回一个 wal.Reader 的引用， 这个结构：承担reader工作的还是这个sr
        //func NewReader(r io.Reader) *Reader {
        //	return &Reader{rdr: r}
        //}
		err = h.loadWAL(wal.NewReader(sr), multiRef, mmappedChunks)
        ...

		h.updateWALReplayStatusRead(i)
	}   
```

h.localWAL 

这个函数先开启了 cpu nums数量的goroutine 做processWalSamples
然后开启了一个goroutine读取数据解析成不同的格式交给decoded。
然后是一个循环处理decoded。如果是sample，就插入到 processWalSamples的输入chanel中。
通过代码分析，prometheus认为最耗时的是processWalSamples的过程。
那么如果想要解决wal replaying的过程，目前应该从两方面考虑啊
- processWalSamples能够提高性能
- processWalSamples是否已经跑满了cpu，cpu利用率达到最高，就是处理任务没有闲置的时候。

```golang
func (h *Head) loadWAL(r *wal.Reader, multiRef map[chunks.HeadSeriesRef]chunks.HeadSeriesRef, mmappedChunks map[chunks.HeadSeriesRef][]*mmappedChunk) (err error) {
    ...
    // n = cpu nums 
    	for i := 0; i < n; i++ {
		processors[i].setup()

		go func(wp *walSubsetProcessor) {
            // 将数据处理到memSeries中 
			unknown := wp.processWALSamples(h)
			unknownRefs.Add(unknown)
			wg.Done()
		}(&processors[i])
	}
    ...
    	go func() {
         		for r.Next() {
                    switch dec.Type(rec) {
			            case record.Series:
                        				decoded <- series
                        ...

                 }
   
        }
        	for d := range decoded {
                ...
                processors[i].input <- 
                ...
            }
}
```
