
代码位置： src/runtime/chan.go

### 结构体
 
##### hchan

src/runtime/chan.go:33 

有元素的数量，类型，类型长度，有发送的index和接收的index，还有发送阻塞队列和接收阻塞队列

```golang 
type hchan struct {

	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

##### waitq 
src/runtime/chan.go:54
```golang
type waitq struct {
	first *sudog
	last  *sudog
}
```

##### sudog

src/runtime/runtime2.go:348

goroutine阻塞队列的封装，有前一个元素和后一个元素的字段

```golang

// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```

### 重要功能

##### 创建chan makechan

src/runtime/chan.go:72

初始化时的内存分配：
- 如果是无缓冲的chan，则只分配hchan结构体的内存
- 如果chan中不包含指针，则申请的存放元素的数组地址和hchan相连
- 如果chan包含指针，单独申请 buf的地址

```golang

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}

```

##### 发送数据 

src/runtime/chan.go:160 

- 阻塞写和非阻塞写 ，如果在select的时候，条件是写入chan，此时就是非阻塞写。其他情况就是阻塞写
- 如果是非阻塞写，并且chan没有关闭，并且chan满了（此时无法写入，那么直接返回false）
    - chan满了有两种情况。1 缓冲区为0的chan是由于此时没有在接收队列上的goroutine 2 缓冲区大于0的chan是由于缓冲区满了
- 如果chan已经是关闭状态，直接panic
- 如果接收等待队列中存在等待的g,取出第一个g，将数据直接发送给它
    - 将数据复制sudog中的一个字段。
	- 唤醒这个g，将其状态从ready变为run
- 如果没有接收等待队列
	- 如果是缓冲区没有满，放入缓冲区后返回
	- 如果是缓冲区满了
		- 如果是非阻塞写，则直接返回false
		- 如果是阻塞写，则将其加入到写队列中，并把这个g状态改为ready

##### 接收数据

src/runtime/chan.go:457

- 如果chan为空
	- 阻塞接收直接crash，退出
	- 非阻塞接收直接返回false
- 非阻塞接收，遇到chan阻塞
	- 如果chan关闭了，再次检查是否是阻塞，如果不是阻塞就接收数据
	- chan没有关闭，直接返回
- 如果遇到chan没有阻塞（就是缓冲区有数据，或者是有发送g在等待）
	- 先从发送队列中取g，从g中国取数据，然后将这个g唤醒
	- 没有的话就从缓冲区拿数据
- 阻塞接收遇到阻塞，
	- 如果chan已经关闭，如果有数据就读数据，没有就不读。返回的时候将selected（标记是否为关闭的chan）设置为1
	- 将自己阻塞在接收队列中国

##### 关闭chan closechan

src/runtime/chan.go:357 

- 如果chan为空，则直接panic
- 如果chan已经关闭了，直接panic
- 加锁
- 将变量closed设置为1
- 释放所有的阻塞的接受者，释放前会检查是否能读到数据
- 释放所有的阻塞的发送者，发送者会panic
- 将所有的发送和接收队列中等待的g状态改为run


