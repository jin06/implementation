### DB struct

  **代码位置** tsdb/db.go:169 

```golang
    type DB struct {
        dir    string
        locker *tsdbutil.DirLocker

        logger         log.Logger
        metrics        *dbMetrics
        opts           *Options
        chunkPool      chunkenc.Pool
        compactor      Compactor
        blocksToDelete BlocksToDeleteFunc

        // Mutex for that must be held when modifying the general block layout.
        mtx    sync.RWMutex
        blocks []*Block

        head *Head

        compactc chan struct{}
        donec    chan struct{}
        stopc    chan struct{}

        // cmtx ensures that compactions and deletions don't run simultaneously.
        cmtx sync.Mutex

        // autoCompactMtx ensures that no compaction gets triggered while
        // changing the autoCompact var.
        autoCompactMtx sync.Mutex
        autoCompact    bool

        // Cancel a running compaction when a shutdown is initiated.
        compactCancel context.CancelFunc
    }
```

#### 数据结构
  
DB主要的变量：
    
    DB 
    -- *tsdbutil.DirLocker
    -- chunkenc.Pool
    -- Compactor
    -- BlocksToDeleteFunc
    -- []*Block
    -- Head

DirLocker

```golang
    type DirLocker struct {
        logger log.Logger

        createdCleanly prometheus.Gauge

        releaser fileutil.Releaser
        path     string
    }
```

最终是体现在一个文件锁上 文件在tsdb/fileutil/flock_unix.go

锁和解锁的代码。最终调用的是如下代码。利用文件锁，文件锁在文件inode上放锁，其他进程可以知道其状态，当进程关闭时文件描述符自动关闭，锁也会自动释放。
```golang
    func (l *unixLock) Release() error {
        if err := l.set(false); err != nil {
            return err
        }
        return l.f.Close()
    }

    func (l *unixLock) set(lock bool) error {
        how := syscall.LOCK_UN
        if lock {
            how = syscall.LOCK_EX
        }
        return syscall.Flock(int(l.f.Fd()), how|syscall.LOCK_NB)
    }
```

其他的结构主要的有，chunk 分配池，压缩，block数组，head。

#### run()

位置 tsdb/db.go:799


启动以后是一个循环，循环内第一select用于接收鬼畜信号。

第二个select三个case
- 第一个每分钟执行一次，先上锁，然后执行 db.reloadBlocks()，然后解锁。之后会往db.compactcc 写入一个数据，以触发第二个case
  - db.reloadBlocks 简单流程
    - 先循循环遍历一边blocks，标记出哪些是要被删除的，然后进行一些列复杂的逻辑后删除掉它们。
- 第二个接收压缩信号
- 第三个接收整个循环的退出信号


```golang
func (db *DB) run() {
	defer close(db.donec)

	backoff := time.Duration(0)

	for {
		select {
		case <-db.stopc:
			return
		case <-time.After(backoff):
		}

		select {
		case <-time.After(1 * time.Minute):
			db.cmtx.Lock()
			if err := db.reloadBlocks(); err != nil {
				level.Error(db.logger).Log("msg", "reloadBlocks", "err", err)
			}
			db.cmtx.Unlock()

			select {
			case db.compactc <- struct{}{}:
			default:
			}
		case <-db.compactc:
			db.metrics.compactionsTriggered.Inc()

			db.autoCompactMtx.Lock()
			if db.autoCompact {
				if err := db.Compact(); err != nil {
					level.Error(db.logger).Log("msg", "compaction failed", "err", err)
					backoff = exponential(backoff, 1*time.Second, 1*time.Minute)
				} else {
					backoff = 0
				}
			} else {
				db.metrics.compactionsSkipped.Inc()
			}
			db.autoCompactMtx.Unlock()
		case <-db.stopc:
			return
		}
	}
}
```

#### func (db *DB) reloadBlocks() error 

run()中有一个比较重要的函数，是删除无效的数据的方法（例如已经过期的）。

执行函数开始的时候先使用锁锁住，使用的是golang的sync.RWMutex. RWMutex 是读写锁，但是其实现有也是基于Mutex。读和写获取锁时使用的方法不同 Lock() 和RLock() . 
这里使用的是写锁。

```golang 
func (db *DB) reloadBlocks() (err error) {
	defer func() {
		if err != nil {
			db.metrics.reloadsFailed.Inc()
		}
		db.metrics.reloads.Inc()
	}()

	// Now that we reload TSDB every minute, there is high chance for race condition with a reload
	// triggered by CleanTombstones(). We need to lock the reload to avoid the situation where
	// a normal reload and CleanTombstones try to delete the same block.
	db.mtx.Lock()
	defer db.mtx.Unlock()

```

然后执行了openBlocks。 查看openBlocks 的实现 ：

```golang 
	loadable, corrupted, err := openBlocks(db.logger, db.dir, db.blocks, db.chunkPool)
	if err != nil {
		return err
	}
```

openBlocks 实现如下： tsdb/db.go:1154

```golang
func openBlocks(l log.Logger, dir string, loaded []*Block, chunkPool chunkenc.Pool) (blocks []*Block, corrupted map[ulid.ULID]error, err error) {
	bDirs, err := blockDirs(dir)
	if err != nil {
		return nil, nil, errors.Wrap(err, "find blocks")
	}

	corrupted = make(map[ulid.ULID]error)
	for _, bDir := range bDirs {
		meta, _, err := readMetaFile(bDir)
		if err != nil {
			level.Error(l).Log("msg", "Failed to read meta.json for a block during reloadBlocks. Skipping", "dir", bDir, "err", err)
			continue
		}

		// See if we already have the block in memory or open it otherwise.
		block, open := getBlock(loaded, meta.ULID)
		if !open {
			block, err = OpenBlock(l, bDir, chunkPool)
			if err != nil {
				corrupted[meta.ULID] = err
				continue
			}
		}
		blocks = append(blocks, block)
	}
	return blocks, corrupted, nil

```

第一步获取block文件所在的所有目录。然后创建map corrupted，开始遍历文件目录数组，获取block目录的meta信息，该信息存在目录的meta.json下，经过处理后返回了一个结构体：

```golang
type BlockMeta struct {
	// Unique identifier for the block and its contents. Changes on compaction.
	ULID ulid.ULID `json:"ulid"`

	// MinTime and MaxTime specify the time range all samples
	// in the block are in.
	MinTime int64 `json:"minTime"`
	MaxTime int64 `json:"maxTime"`

	// Stats about the contents of the block.
	Stats BlockStats `json:"stats,omitempty"`

	// Information on compactions the block was created from.
	Compaction BlockMetaCompaction `json:"compaction"`

	// Version of the index format.
	Version int `json:"version"`
}
// BlockMetaCompaction holds information about compactions a block went through.
type BlockMetaCompaction struct {
	// Maximum number of compaction cycles any source block has
	// gone through.
	Level int `json:"level"`
	// ULIDs of all source head blocks that went into the block.
	Sources []ulid.ULID `json:"sources,omitempty"`
	// Indicates that during compaction it resulted in a block without any samples
	// so it should be deleted on the next reloadBlocks.
	Deletable bool `json:"deletable,omitempty"`
	// Short descriptions of the direct blocks that were used to create
	// this block.
	Parents []BlockDesc `json:"parents,omitempty"`
	Failed  bool        `json:"failed,omitempty"`
}
```

然后就是一个一个的对比文件中的block和db这个结构体（内存）中的block，如果内存中没有就调用OpenBlock函数添加进来。
现在这个函数就返回了
	
	- blocks 所有的block
	- corrupted 内存中没有的block，并且调用open打开失败的。这些可能时失效的block

回到reloadBlocks函数，接下来执行的是：

```golang
	deletableULIDs := db.blocksToDelete(loadable)
	deletable := make(map[ulid.ULID]*Block, len(deletableULIDs))
```

调用了db.blocksToDelete(),看名字知道是要删除的block的uild的数组，然后创建一个map，大小就是要删除的block的数量。
函数的默认实现是 tsdb/db.go:1191, 如果在上次压缩时内没有样本数据，这次删除。如果超过了最大保留时间，删除。如果大小超过了上限则删除。这三种情况。



```golang
// deletableBlocks returns all currently loaded blocks past retention policy or already compacted into a new block.
func deletableBlocks(db *DB, blocks []*Block) map[ulid.ULID]struct{} {
	deletable := make(map[ulid.ULID]struct{})

	// Sort the blocks by time - newest to oldest (largest to smallest timestamp).
	// This ensures that the retentions will remove the oldest  blocks.
	sort.Slice(blocks, func(i, j int) bool {
		return blocks[i].Meta().MaxTime > blocks[j].Meta().MaxTime
	})

	for _, block := range blocks {
		if block.Meta().Compaction.Deletable {
			deletable[block.Meta().ULID] = struct{}{}
		}
	}

	for ulid := range BeyondTimeRetention(db, blocks) {
		deletable[ulid] = struct{}{}
	}

	for ulid := range BeyondSizeRetention(db, blocks) {
		deletable[ulid] = struct{}{}
	}

	return deletable
}
```

接下来， 遍历内存中的blocks，如果存在，则放入到deletable的map中，遍历block的父block，如果存在父block，则父block被保留，即将deletable 中的父block的值设置为nil。

```golang
	// Mark all parents of loaded blocks as deletable (no matter if they exists). This makes it resilient against the process
	// crashing towards the end of a compaction but before deletions. By doing that, we can pick up the deletion where it left off during a crash.
	for _, block := range loadable {
		if _, ok := deletableULIDs[block.meta.ULID]; ok {
			deletable[block.meta.ULID] = block
		}
		for _, b := range block.Meta().Compaction.Parents {
			if _, ok := corrupted[b.ULID]; ok {
				delete(corrupted, b.ULID)
				level.Warn(db.logger).Log("msg", "Found corrupted block, but replaced by compacted one so it's safe to delete. This should not happen with atomic deletes.", "block", b.ULID)
			}
			deletable[b.ULID] = nil
		}
	}
```

如果此时corrupted仍然大于0，则将打开的错误聚合并返回。函数到此结束。

否则接着执行, 这里的操作保留了旧的blocks，然后将没有删除的blocks赋值给当前的blocks。然后删除了blocks

```golang
	// All deletable blocks should be unloaded.
	// NOTE: We need to loop through loadable one more time as there might be loadable ready to be removed (replaced by compacted block).
	for _, block := range loadable {
		if _, ok := deletable[block.Meta().ULID]; ok {
			deletable[block.Meta().ULID] = block
			continue
		}

		toLoad = append(toLoad, block)
		blocksSize += block.Size()
	}
	db.metrics.blocksBytes.Set(float64(blocksSize))

	sort.Slice(toLoad, func(i, j int) bool {
		return toLoad[i].Meta().MinTime < toLoad[j].Meta().MinTime
	})
	if !db.opts.AllowOverlappingBlocks {
		if err := validateBlockSequence(toLoad); err != nil {
			return errors.Wrap(err, "invalid block sequence")
		}
	}

	// Swap new blocks first for subsequently created readers to be seen.
	oldBlocks := db.blocks
	db.blocks = toLoad

	blockMetas := make([]BlockMeta, 0, len(toLoad))
	for _, b := range toLoad {
		blockMetas = append(blockMetas, b.Meta())
	}
	if overlaps := OverlappingBlocks(blockMetas); len(overlaps) > 0 {
		level.Warn(db.logger).Log("msg", "Overlapping blocks found during reloadBlocks", "detail", overlaps.String())
	}

	// Append blocks to old, deletable blocks, so we can close them.
	for _, b := range oldBlocks {
		if _, ok := deletable[b.Meta().ULID]; ok {
			deletable[b.Meta().ULID] = b
		}
	}
	if err := db.deleteBlocks(deletable); err != nil {
		return errors.Wrapf(err, "delete %v blocks", len(deletable))
	}
	return nil
}
```

删除的代码如下，它先把要删除的文件重命名然后再删除，目的可能时防止删除过程中出现down机，导致文件损坏，损坏的文件又没有被标记为删除造成逻辑上的错误。

```golang
func (db *DB) deleteBlocks(blocks map[ulid.ULID]*Block) error {
	for ulid, block := range blocks {
		if block != nil {
			if err := block.Close(); err != nil {
				level.Warn(db.logger).Log("msg", "Closing block failed", "err", err, "block", ulid)
			}
		}

		toDelete := filepath.Join(db.dir, ulid.String())
		if _, err := os.Stat(toDelete); os.IsNotExist(err) {
			// Noop.
			continue
		} else if err != nil {
			return errors.Wrapf(err, "stat dir %v", toDelete)
		}

		// Replace atomically to avoid partial block when process would crash during deletion.
		tmpToDelete := filepath.Join(db.dir, fmt.Sprintf("%s%s", ulid, tmpForDeletionBlockDirSuffix))
		if err := fileutil.Replace(toDelete, tmpToDelete); err != nil {
			return errors.Wrapf(err, "replace of obsolete block for deletion %s", ulid)
		}
		if err := os.RemoveAll(tmpToDelete); err != nil {
			return errors.Wrapf(err, "delete obsolete block %s", ulid)
		}
		level.Info(db.logger).Log("msg", "Deleting obsolete block", "block", ulid)
	}

	return nil
}
```

#### 扩展

> 在组织block时使用了一个第三方库去做block的唯一标识符。 https://github.com/oklog/ulid
>  
> 它是 https://github.com/ulid/javascript 的golang的实现。
> 它的作用时替代GUID和UUID的功能。根据官网介绍的，GUID和UUID的缺陷是：
>   - 128 bits编码效率不是最高
>   - UUID V1/V2 在很多场景下不适用，因为它需要访问一个唯一的稳定的MAC地址？
>   - UUID V3/V4 需要一个唯一的种子生成分布式的ID值，这样会造成大量的碎片
>   - UUID V4 除了随机没有提供其他信息，也会造成大量的碎片。
> 
> 大概的意思流程，它最终生成一个容量为16的字节数组。那么总的位数就是16*8=128位。其中前32位+16位表示时间，后面是随机数，随机数部分分为三部分生成。可以从生成的id中识别到时间。
> 
> 实现细节不用管，可以知道prometehus使用该库生成block的唯一标识


