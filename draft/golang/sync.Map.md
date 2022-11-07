
- sync.Map 
- src/sync/map.go
- 一个线程安全的map

### sync.Map 中的注释对其实现原理的介绍

src/sync/map.go:11

- sync.Map 是线程安全的，可以放心使用
- sync.Map 是有其特定适用场景，一般情况下还是使用普通的map加分段锁的方式实现线程安全
- sync.Map 适合的场景
    - 读多写少
    - 不连贯的键
- 对外提供的方法，主要两种操作 读 写
    - Load LoadAndDelete LoadOrStore 
    - Delete LoadAndDelete Store

### Map 结构

```golang 
type Map struct {
	mu Mutex

	// read contains the portion of the map's contents that are safe for
	// concurrent access (with or without mu held).
	//
	// The read field itself is always safe to load, but must only be stored with
	// mu held.
	//
	// Entries stored in read may be updated concurrently without mu, but updating
	// a previously-expunged entry requires that the entry be copied to the dirty
	// map and unexpunged with mu held.
	read atomic.Pointer[readOnly]

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
	dirty map[any]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
	misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[any]*entry
	amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted, and either m.dirty == nil or
	// m.dirty[key] is e.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p atomic.Pointer[any]
}
```

### 读方法

##### Load

- 先读readonly的map，此时是不加锁的
    - 如果有读到数据
        - 如果read map 和 write map 有不一致（全局的不一致），此时加一个锁， 则读write map
            - 如果读write map 读到了数据，返回该数据，并且将Map中miss增加
                - 如果miss数量大于等于 write map中的元素数量，则触发将write 更新到read map 中
    - 如果读到了数据，则返回数据

##### Store

- 先去 read map 读数据
    - 如果存在数据， 试着存储新数据，存储数据使用的是原子操作 CompareAndSwap
- 如果read map 没有数据 或尝试写入数据失败 （原因一个是元素已经被删除，或者是存在并发写入，原子写入失败）
    - 加锁
    - 再次读取 read map 数据
        - 如果read map 中存在数据
            - 如果元素没有被标记删除， 则直接写入到 write map中
            - 将新的值写入，这就是store 的更新
        - 如果 read map 没有数据
            - 如果 write map中有数据 
                - 将数据写入到write map中
            - 如果 write map 中没有数据 
                - 如果write map中的数据 read map中都有 ，此时会干两件事
                    - 触发一次检查，检查write map是否为空，如果为空则将read map中的数据全部更新到 write map中
                    - 将 amended 改为true， write map中包含了 read map中没有的元素
                - 最后将新的元素添加到 write map中


逻辑总结，调用store存储一个kv数据时，可能遇到的情况：
1. Map中没有这个 key 对应的 value
     - 此时会加锁，然后触发一个动作——是否需要将readmap中的数据复制到dirtymap。 
     - 向dirtymap添加数据 。
     - 所有完成后呢，ammended现在为 true， 即readmap中有脏数据，如果readmap中有脏数据，那么Map读数据的时候就要触发检查。
2. readmap中不存在，dirtymap中存在
    - 直接修改dirtymap中的对应的元素即可，readmap不用管
3. readmap中存在
    - 更新这个readmap中的数据，其中使用了 load 和 compareAndSwap 原子方法，保证更新过程不会出现并发错误，返回结果
    - 如果readmap中存在的这个数据被标记为删除，也会返回错误
    - 如果向readmap更新失败，此时加一个锁
        - 然后将删除标记为不删除，将指针也同时赋值给dirtymap
        - 直接修改key对应的元素的值，（这个版本中，dirtymap和readmap对应的底层的元素相同，各自保留了引用，改一个另一个就成功了）

##### 