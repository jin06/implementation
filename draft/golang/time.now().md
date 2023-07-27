
## 遇到的问题

- 写单元测试调用 time.Now() 方法，每次返回的结构都是相同的。如果修改单元测试的代码，当修改单元测试代码时时间才会变化
- 环境
  - macbook 
  - golang 1.20.5 darwin/amd64 


## time.Now()

- go最终实现的代码 在 go/src/runtime/timestub.go:15
 
```
//go:linkname time_now time.now
func time_now() (sec int64, nsec int32, mono int64) {
	sec, nsec = walltime()
	return sec, nsec, nanotime()
}
 ```

- 展开 walltime()
 
```
//go:nosplit
//go:cgo_unsafe_args
func walltime() (int64, int32) {
	var t timespec
	libcCall(unsafe.Pointer(abi.FuncPCABI0(walltime_trampoline)), unsafe.Pointer(&t))
	return t.tv_sec, int32(t.tv_nsec)
}
func walltime_trampoline()

```

- amd64架构的执行 这段代码, 最终调用 clock_gettime
 
```
TEXT runtime·walltime_trampoline(SB),NOSPLIT,$0
 // 
	PUSHQ	BP			// make a frame; keep stack aligned
	MOVQ	SP, BP
	MOVQ	DI, SI			// arg 2 timespec
	MOVL	$CLOCK_REALTIME, DI	// arg 1 clock_id
	CALL	libc_clock_gettime(SB)
	POPQ	BP
	RET

```
