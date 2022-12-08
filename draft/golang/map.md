
## 简介
代码地址 src/runtime/map.go

#### 数据结构

- map 对应的结构为 hmap
- bucket中存放元素，可以存放8个元素，key经过哈希后存放在对应的桶中
- 如果桶超过容量限制，则使用extra来装填多出的冲突的桶
- mapextra 管理溢出桶，其中只有hmap中桶满了，才会使用溢出桶
- bmap是桶的结构
- hiter 迭代用途
- evacDst 扩容时候使用，就buketes到新buckets
```golang
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}


// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}


// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

// A hash iteration structure.
// If you modify hiter, also change cmd/compile/internal/reflectdata/reflect.go
// and reflect/value.go to match the layout of this structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/compile/internal/walk/range.go).
	elem        unsafe.Pointer // Must be in second position (see cmd/compile/internal/walk/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}

// evacDst is an evacuation destination.
type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

```

### makemap

src/runtime/map.go:304

```golang 
func makemap(t *maptype, hint int, h *hmap) *hmap {
    ...
    // 生成哈希的随机种子
    	h.hash0 = fastrand()
    ...
    // 初始化桶的数量
    B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
    ...
    // 初始化
    if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
}
```
 
 src/runtime/map.go:345

 ```golang 
 func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {}
 ```

 makeBucketArray 
根据元素类型，容量大小初始化若干个桶

### map扩容


扩容函数 src/runtime/map.go:1041 

扩容时机，桶的数量和已经有的元素数量，和一个常量组成了一个公式，计算出是否是否需要扩容
代码,调用的时候是 count+1，也就是当前map中元素数量加1.
也就是count / 2^B > 13/ 2  我这里看到的版本 loadFactorNum 是13 。 
然后也就是 举个例子 B= 2 桶的数量是16，map中元素再加1大于 16*6.5 时扩容。（桶等于16时，桶中可以装 16*8 个元素）
```golang 
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

扩容的过程，扩容是一步一步扩容的，先生成一个新的buckets，然后再每次增删map元素的时候会进行重新的hash，将键值对分到新的桶中，删除旧的桶中的元素

### map迭代

迭代器注意初始化的时候会产生一个随机数，然后确定迭代开始的位置，开始的数组

迭代涉及到了当前的buckets数组，old buckets数组和溢出的元素，代码太复杂，大概逻辑应该是先遍历当前


### mapaccess1

val := m[i] 语法对应的runtime运行时函数。

golang编译时会将相应语法替换成这个函数。

注意：
- 如果map中不存在，则返回一个0值
- 返回的指针会保持整个map的存活，不会被gc回收。

执行过程：
- 如果map中没有key对应的值，则返回零值
- 如果此时有另一个goroutine在写，则发生致命错误。程序退出。
- 计算哈希值。
- 根据哈希值计算bucket
- 计算oldBucket，判断要查找的值落在了那个bucket中 （这个涉及到map的一个扩容，map扩容后，不是一次性迁移老的bucket，而是慢慢迁移，所以有一段时间，oldBucket和新的bucket同时存在）
- 遍历当前的bucket，然后判断比对key值是否为查询的值，如果是则返回。函数退出
- 没有查询到任何值匹配，返回零值。