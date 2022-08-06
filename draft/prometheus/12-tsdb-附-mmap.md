
tsdb wal replay 用到了mmap。 位置在 tsdb/fileutil/mmap_unix.go:25


    func mmap(f *os.File, length int) ([]byte, error) {
	    return unix.Mmap(int(f.Fd()), 0, length, unix.PROT_READ, unix.MAP_SHARED)
    }

最终使用的是 开源package  golang.org/x/sys/unix

```golang

//vendor/golang.org/x/sys/unix/syscall_bsd.go:611

func Mmap(fd int, offset int64, length int, prot int, flags int) (data []byte, err error) {
	return mapper.Mmap(fd, offset, length, prot, flags)
}

//vendor/golang.org/x/sys/unix/syscall_unix.go:108

func (m *mmapper) Mmap(fd int, offset int64, length int, prot int, flags int) (data []byte, err error) {
...
	addr, errno := m.mmap(0, uintptr(length), prot, flags, fd, offset)
    // 最终返回的[]byte 起始地址就是addr指向的位置 mmap最终是系统调用mmap
... 
}




```


#### 系统调用  mmap 

Memory-mapped file support. 
用户程序读取文件常规流程涉及到内核到用户空间的拷贝，这是操作系统中内存管理部分。
简单说明 mmap提供一种机制，直接映射到内核空间的内存中，用户程序通过这个引用可以直接读取到文件，减少了内核空间和用户空间之间的内存拷贝，提升了性能。释放空间的函数是 munmap

函数的定义

    void* mmap ( void * addr , size_t len , int prot , int flags , int fd , off_t offset )


prot 指定共享内存的访问权限，prometheus中使用的是read权限 ， flags 选择的是 MAP_SHARED，顾名思义，这段内存空间对这段内存建立了mmap的进程都是共享的，修改对大家都是可见的。
