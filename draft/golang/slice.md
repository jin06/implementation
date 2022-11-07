
代码主要位置 src/runtime/slice.go

### 结构

```golang
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

### slice 增长 

src/runtime/slice.go:157

对应的扩容函数是 growslice 

传入的参数
- oldPrt 旧slice对应的数组地址
- num 本次增加的数组元素数量
- newLen 扩容后的长度 oldLen + num 
- et 数组元素类型

输入的值
- 输出一个新的slice

```golang
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice
```

最终调用分配的数组使用的函数 是p = mallocgc(capmem, et, true)

其中capmem通过一些列规则确定
- newcap 确定
    - 如果neLen大于两倍的oldCap ， 则newcap = newLen
    - 如果oldCap小于 256 ， newcap = 2 * oldCap。 否则newcap通过公式赋值，newcap = 1.25*newcap + 192 ，知道necap大于newLen为止
- capmem是由newcap确定

最后是移动旧元素到新的slice对应的数组中 memmove(p, oldPtr, lenmem)