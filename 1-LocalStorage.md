
### LocalStorage 启动

在main函数中，1118行，是启动的封装：

    func openDBWithMetrics(dir string, logger log.Logger, reg prometheus.Registerer, opts *tsdb.Options, stats *tsdb.DBStats) (*tsdb.DB, error) 	
    db, err := tsdb.Open(
		dir,
		log.With(logger, "component", "tsdb"),
		reg,
		opts,
		stats,
	)
    .....
  }

主要调用了tsdb包下的Open函数。最终执行 tsdb/db.go 613行的函数 open。
- 传入的参数主要的作用
    - 传入的参数dir 是localstorage 的path。
    - Registerer 和 *DBStats 是用来监控自己的
    - *Options 设置的参数 WAL block大小，WAL是否压缩，数据保留时间等
    - rngs 是调用open()的函数传入的，调用函数为Open，使用validateOpts初始化一个值，位置在tsdb/db.go:579，最终调用的函数在 tsdb/compact.go:42
        - 其中参数 minSize在最上层传入的名字是 MinBlockDuration 。
        - 最终结果是初始化了一个大小为10的[]int64的数组，第一个数字是10，然后依次递增3倍。

        ```golang
            rngs = ExponentialBlockRanges(opts.MinBlockDuration, 10, 3)
        ```
        ```golang 
            func ExponentialBlockRanges(minSize int64, steps, stepSize int) []int64 {
                ranges := make([]int64, 0, steps)
                curRange := minSize
                for i := 0; i < steps; i++ {
                    ranges = append(ranges, curRange)
                    curRange = curRange * int64(stepSize)
                }

                return ranges
            }            
        ```
-  首先创建了localstorage目录 tsdb/db.go:614
-  对rngs参数进一步处理，过  tsdb/db.go:625
-  一些兼容老代码的操作。tsdb/db.go:632
-  wal 操作 tsdb/db.go:636。 tsdb/wal.go:1232
    - ahead log 的管理
        - 创建或者打开ahead log tsdb/wal.go:175。 
        - 其中有一项操作是定时同步数据 位置在tsdb/wal.go:700, 将segment文件在内存的变化同步到磁盘. interval 它给固定了1分钟。没分钟会同步一次
        ```golang
            func (w *SegmentWAL) run(interval time.Duration) {
                var tick <-chan time.Time

                if interval > 0 {
                    ticker := time.NewTicker(interval)
                    defer ticker.Stop()
                    tick = ticker.C
                }
                .......

                for {
                    ......
                    case <-tick:
                        if err := w.Sync(); err != nil {
                            level.Error(w.logger).Log("msg", "sync failed", "err", err)
                        }                    
                }
            }        
        ```

    - 启动时有一个数据修复动作，从日志到localStorage,经过一系列调用，最终在走到这里 tsdb/wal.go:869
        - 简单说就是从日志中读数据，解析成db需要存储的结构，写入到本地的db文件中。
        - 其中 tsdb/wal.go:227 repairingWALReader 这个结构负责读文件读取数据，简单的处理成字节数组。最终是由 WAL这个对象负责写入到本地db文件，WAL的位置在 tsdb/wal/wal.go:176。 wal的实现放在后面在看，这部分是关于prometehus怎么在硬盘组织它的数据库文件，是关键部分。

```golang
    func open(dir string, l log.Logger, r prometheus.Registerer, opts *Options, rngs []int64, stats *DBStats) (_ *DB, returnedErr error) {
        if err := os.MkdirAll(dir, 0o777); err != nil {
            return nil, err
        }
        if l == nil {
            l = log.NewNopLogger()
        }
        if stats == nil {
            stats = NewDBStats()
        }

        for i, v := range rngs {
            if v > opts.MaxBlockDuration {
                rngs = rngs[:i]
                break
            }
        }

        // Fixup bad format written by Prometheus 2.1.
        if err := repairBadIndexVersion(l, dir); err != nil {
            return nil, errors.Wrap(err, "repair bad index version")
        }

        walDir := filepath.Join(dir, "wal")

        // Migrate old WAL if one exists.
        if err := MigrateWAL(l, walDir); err != nil {
            return nil, errors.Wrap(err, "migrate WAL")
        }
        for _, tmpDir := range []string{walDir, dir} {
            // Remove tmp dirs.
            if err := removeBestEffortTmpDirs(l, tmpDir); err != nil {
                return nil, errors.Wrap(err, "remove tmp dirs")
            }
        }

        db := &DB{
            dir:            dir,
            logger:         l,
            opts:           opts,
            compactc:       make(chan struct{}, 1),
            donec:          make(chan struct{}),
            stopc:          make(chan struct{}),
            autoCompact:    true,
            chunkPool:      chunkenc.NewPool(),
            blocksToDelete: opts.BlocksToDelete,
        }
        defer func() {
            // Close files if startup fails somewhere.
            if returnedErr == nil {
                return
            }

            close(db.donec) // DB is never run if it was an error, so close this channel here.

            returnedErr = tsdb_errors.NewMulti(
                returnedErr,
                errors.Wrap(db.Close(), "close DB after failed startup"),
            ).Err()
        }()

        if db.blocksToDelete == nil {
            db.blocksToDelete = DefaultBlocksToDelete(db)
        }

        var err error
        db.locker, err = tsdbutil.NewDirLocker(dir, "tsdb", db.logger, r)
        if err != nil {
            return nil, err
        }
        if !opts.NoLockfile {
            if err := db.locker.Lock(); err != nil {
                return nil, err
            }
        }

        ctx, cancel := context.WithCancel(context.Background())
        db.compactor, err = NewLeveledCompactorWithChunkSize(ctx, r, l, rngs, db.chunkPool, opts.MaxBlockChunkSegmentSize, nil)
        if err != nil {
            cancel()
            return nil, errors.Wrap(err, "create leveled compactor")
        }
        db.compactCancel = cancel

        var wlog *wal.WAL
        segmentSize := wal.DefaultSegmentSize
        // Wal is enabled.
        if opts.WALSegmentSize >= 0 {
            // Wal is set to a custom size.
            if opts.WALSegmentSize > 0 {
                segmentSize = opts.WALSegmentSize
            }
            wlog, err = wal.NewSize(l, r, walDir, segmentSize, opts.WALCompression)
            if err != nil {
                return nil, err
            }
        }

        headOpts := DefaultHeadOptions()
        headOpts.ChunkRange = rngs[0]
        headOpts.ChunkDirRoot = dir
        headOpts.ChunkPool = db.chunkPool
        headOpts.ChunkWriteBufferSize = opts.HeadChunksWriteBufferSize
        headOpts.ChunkWriteQueueSize = opts.HeadChunksWriteQueueSize
        headOpts.StripeSize = opts.StripeSize
        headOpts.SeriesCallback = opts.SeriesLifecycleCallback
        headOpts.EnableExemplarStorage = opts.EnableExemplarStorage
        headOpts.MaxExemplars.Store(opts.MaxExemplars)
        headOpts.EnableMemorySnapshotOnShutdown = opts.EnableMemorySnapshotOnShutdown
        if opts.IsolationDisabled {
            // We only override this flag if isolation is disabled at DB level. We use the default otherwise.
            headOpts.IsolationDisabled = opts.IsolationDisabled
        }
        db.head, err = NewHead(r, l, wlog, headOpts, stats.Head)
        if err != nil {
            return nil, err
        }

        // Register metrics after assigning the head block.
        db.metrics = newDBMetrics(db, r)
        maxBytes := opts.MaxBytes
        if maxBytes < 0 {
            maxBytes = 0
        }
        db.metrics.maxBytes.Set(float64(maxBytes))

        if err := db.reload(); err != nil {
            return nil, err
        }
        // Set the min valid time for the ingested samples
        // to be no lower than the maxt of the last block.
        blocks := db.Blocks()
        minValidTime := int64(math.MinInt64)
        if len(blocks) > 0 {
            minValidTime = blocks[len(blocks)-1].Meta().MaxTime
        }

        if initErr := db.head.Init(minValidTime); initErr != nil {
            db.head.metrics.walCorruptionsTotal.Inc()
            level.Warn(db.logger).Log("msg", "Encountered WAL read error, attempting repair", "err", initErr)
            if err := wlog.Repair(initErr); err != nil {
                return nil, errors.Wrap(err, "repair corrupted WAL")
            }
        }

        go db.run()

        return db, nil
    }

```

localStorage组件启动过程结束。


