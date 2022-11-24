
 src/sync/mutex.go:34 

- 是一个互斥锁
- 0值代表没有上锁
- 第一次使用后就不要在复制这个锁了
- 

 ```golang
 type Mutex struct {
	state int32
	sema  uint32
}
 ```


 ### Lock

- 如果m已经上锁，则调用的线程阻塞直到锁释放


 ```golang
 // Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
    // 原子命令，如果state的值等于0，则将它修改为1，表示上锁成功
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
    // lock已经锁定了，进入到m.lockSlow 处理流程
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
 ```

 如果已经上锁了，则goroutine执行的操作

 - 判断能否自旋，应该就是goroutine一直占据cpu，进行忙等待，如果满足以下条件则进行自旋
    - 自旋时间短
    - 多核机器
    - GOMAXPROCS 参数大于1 
    - 至少有一个其他的正在运行的P
    - 本地的runq是空的
    

### Unlock

- 释放锁 并唤醒一个等待的goroutine。如果是饥饿模式直接唤醒这个goroutine

```golang
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```
