

# 哈希碰撞
哈希碰撞导致同一个桶中可能存在多个元素，有多种方式可以避免哈希碰撞，一般有两种主要的策略：拉链法和开放寻址法。
+ 拉链法。将同一个桶中的元素通过链表的形式进行链接。
	+ 随着桶中的元素增加，可以不断链接新的元素，不用预先为元素分配内存
	+ 不足之处在于，需要存储额外的指针用于链接元素，增加整个哈希表的大小；同时由于链表存储的地址不连续，无法高校利用CPU高速缓存
+ 开放寻址法。所有元素存储在桶数组中。碰撞时，会按照某种探测策略找到可用的位置

**go中的哈希表采用的是优化的拉链法，每一个桶中存储了8个元素用于加速访问**

# 基本用法

## 声明与初始化

+ 声明
```go
var ma map[int]int
// ma为nil map
// 对nil map赋值：panic( assignment to entry in nil map)
// 对nil map访问： 返回类型的零值
```

+ 通过make函数初始化。第二个参数表示初始化创建map的长度，为空时默认是0
```go
ma:=make(map[int]int)
ma:=map[int]int{
	1:1,
	2:2,
}
ma:=map[int]int{1:1,2:2}
```
## 访问

```go
// 第一种方式
v:=ma[1]
// 第二种方式，第二个参数表示当前正在访问的key是否存在
v,ok:=ma[1]
```

## 赋值
```go
ma[1]=2
```

delete是go中的关键字，可以多次删除相同的key不会报错
```go
delete(ma,1)
```


## key的可比较性

+ 布尔值是可以比较的
+ 整数值是可以比较的
+ 浮点值是可以比较的
+ 复数值是可以比较的
+ 字符串值是可以比较的
+ 指针值是可以比较的。如果两个指针指向相同的变量，或者两个指针的值均为nil，则它们相等
+ 通道值是可以比较的。如果两个通道是由相同的make函数调用创建的，或者两个值都为nil，则它们相等
+ 接口值是可以比较的。如果两个接口值具有相同的动态类型和相等的动态值，或者两个接口都为nil，则它们相等
+ 如果结构体的所有字段都是可比较的，它们的值是可比较的
+ 如果数组元素类型的值是可比较的，则数组值可比较。如果两个数组对应的元素相等，则他们相等
+ slice、map、func不可以比较
## 并发冲突

+ map并不支持并发的读写，会fatal error: concurrent map read and map write
```go
func main() {
	ma := make(map[int]int)
	go func() {
		for {
			_ = ma[3]
		}
	}()	
	go func() {
		for{
			ma[3]=3
		}
	}()
	time.Sleep(time.Second*5)
}

// fatal error: concurrent map read and map write


func main() {
	ma := make(map[int]int)
	go func() {
		for {
			ma[3]=2
		}
	}()
	go func() {
		for{
			ma[3]=3
		}
	}()
	time.Sleep(time.Second*5)
}

// fatal error: concurrent map writes
```

+ map只支持并发读取操作
```go
func main() {
	ma := make(map[int]int)
	go func() {
		for {
			_=ma[3]
		}
	}()
	
	go func() {
		for{
			_=ma[2]
		}
	}()
	time.Sleep(time.Second*5)
}
```

**go为什么不支持并发的读写**

官方文档的解释是：
map不需要从多个goroutine安全访问，在实际情况下，map可能是某些已经同步的较大数据结构或计算的一部分。因此，要求所有map操作都互斥将减慢大多数程序的速度，而只会增加少数程序的安全性，即go只支持并发读取的原因是保证大多数场景下的查找效率

# 底层原理
`runtime/map.go`

## 源码给出的说明

本文件实现go语言的map类型

map本质上是一个哈希表。其数据存储在buckets数组中，每个bucket最多包含8个key/elem对
哈希值的低位用于选择桶，每个桶会存储部分哈希高位用于区分桶内不同条目

当一个桶映射超过8个键时，会通过链表方式串联溢出桶(overflow buckets)

哈希表扩容时，会分配两倍大小的新桶数组，并通过渐进式迁移将旧桶数据复制到新桶数组

map迭代器会遍历整个桶数组，按照遍历顺序返回键：
先桶序号 --  再溢出链序号 -- 最后桶内索引
为保证迭代语义，从不移动桶内的键(否则可能返回重复键)，{即，key/elem在旧桶条目的中位置，在新桶条目同样的位置}

扩容期间，迭代器继续遍历旧桶数组，但需要检查当前bucket是否已经被evacuated到新数组

**关于负载因子的选择**
过大，会产生大量溢出桶
过小，会浪费空间

以下是不同负载因子的测试数据(64位系统，8字节键值)

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
| ---------- | --------- | ----------- | -------- | --------- |
| 4.00       | 2.13      | 20.77       | 3.00     | 4.00      |
| 4.50       | 4.05      | 17.30       | 3.25     | 4.50      |
| 5.00       | 6.85      | 14.77       | 3.50     | 5.00      |
| 5.50       | 10.55     | 12.94       | 3.75     | 5.50      |
| 6.00       | 15.27     | 11.67       | 4.00     | 6.00      |
| 6.50       | 20.90     | 10.79       | 4.25     | 6.50      |
| 7.00       | 27.14     | 10.15       | 4.50     | 7.00      |
| 7.50       | 34.03     | 9.73        | 4.75     | 7.50      |
| 8.00       | 41.10     | 9.40        | 5.00     | 8.00      |
+ `%overflow`。存在溢出桶的桶占比(百分比)
+ `bytes/entry`。每个键值对的内存开销(字节)
+ `hitprobe`。查找存在键时需要检查的条目数
+ `missprobe`。查找不存在的键时需要检查的条目数

表格中的数据对应的是即将扩容前的哈希表满载的状态
实际使用中的哈希表负载通常会更低一些。

## hmap结构体

一些定义好的常量

```go
const (
	// 单个存储桶(bucket)最多可以容纳的键值对数量
	bucketCntBits = abi.MapBucketCountBits // 3
	bucketCnt = abi.MapBucketCount // 8
	
	
	// 触发扩容的存储桶平均最大负载为buckent*13/16(约80%满容量)
	// 由于最小对齐规则限制，buckent已知至少为8
	// 表示为loadFactorNum / loadFactorDen形式以便进行整数运算
	loadFactorDen = 2
	loadFactorNum = loadFactorDen * bucketCnt * 13 / 16 // 13
	  
	// 内联存储(而非为每个元素单独分配内存)的最大键/元素大小
	// 必须能容纳在uint8类型范围内
	// 快速处理版本无法处理大元素--cmd/complie/internal/gc/walk.go
	// 快速版本的阈值大小必须等于此elem值
	maxKeySize = abi.MapMaxKeyBytes //128
	maxElemSize = abi.MapMaxElemBytes //128
	
	
	// 数据偏移量(data offset)理论上应该等于bmap结构体大小，但必须满足正确的内存对齐要求
	// 对于amd64p32架构，即使指针为32位，仍然需要保持64位对齐
	// bmap结构体只有一个字段tophash [8]uint8
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)
	
	// tophash可能的取值，保留了一些特殊标记值
	// 每个桶(包括溢出桶，若有)中的条目要么全部处于evacuated状态，要么全不处于该状态
	// evacuate()方法执行期间例外，此方法仅在map写入时触发，故而其它操作无法在此期间观测到map状态
	emptyRest = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
	emptyOne = 1 // this cell is empty
	evacuatedX = 2 // key/elem is valid. Entry has been evacuated to first half of larger table.
	evacuatedY = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash = 5 // minimum tophash for a normal filled cell.
	
	// flags
	iterator = 1 // there may be an iterator using buckets
	oldIterator = 2 // there may be an iterator using oldbuckets
	hashWriting = 4 // a goroutine is writing to the map
	sameSizeGrow = 8 // the current map growth is to a new map of the same size
	
	// sentinel bucket ID for iterator checks
	noCheck = 1<<(8*goarch.PtrSize) - 1
)
```

**tophash的可能取值**

| tophash        | Number | Comment                             |
| -------------- | ------ | ----------------------------------- |
| emptyRest      | 0      | 当前cell为空，并且更高索引位置以及溢出链中均不存在非空cell   |
| emptyOne       | 1      | 当前cell是空的                           |
| evacuatedX     | 2      | key/element是有效的，该entry已经被迁移到扩容表的前半段 |
| evacuatedY     | 3      | key/element是有效的，该entry已经被迁移到扩容表的后半段 |
| evacuatedEmpty | 4      | 当前cell是空的，当前桶已经完成迁移                 |
| minTopHash     | 5      | 普通非空cell的最小tophash值                 |
**比较重要的两个flags字段**

| flag         | Number | Comment     |
| ------------ | ------ | ----------- |
| hashWriting  | 4      | 有协程正在写入该map |
| sameSizeGrow | 8      | 等量扩容        |


**可以看到hmap中没有锁，所以map不是并发读写安全的**

```go
// A header for a Go map.
type hmap struct {
	// hmap的格式也同时在cmd/complie/internal/reflectdata/reflect.go
	// 请确保此处定义与编译器的定义保持同步
	
	count int // 元素个数；放在第一个位置，供len()函数使用
	flags uint8 //标识当前map是正在读还是写
	
	B uint8 // 桶数量以2为底的对数(最多可容纳loadFactor*2^B个元素)
	noverflow uint16 // 大约的溢出桶数量，详情参阅incrnoverflow
	hash0 uint32 // hash seed
	
	buckets unsafe.Pointer // 含有2^B个bucket的数组，如果count==0可能为nil
	oldbuckets unsafe.Pointer // 大小为一半的旧buckets数组，仅在扩容时非nil
	nevacuate uintptr // 迁移进度计数器，小于此值的桶已经完成迁移
	
	extra *mapextra // 选项字段
}
```

**mapextra**


```go
// mapextra holds fields that are not present on all maps.
// mapextra结构体包含的字段，maps不一定都包含
type mapextra struct {
	// 如果键和值都不包含指针且是内联的，则将桶类型标记为不包含指针，这样可以避免扫描此类映射
	// bmap.overflow是一个指针，为了保持溢出桶存活，将所有溢出桶的指针存储在hmap.extra.overflow和hmap.extra.oldoverflow中
	// overflow和oldoverflow仅在键和值不包含指针时使用
	// overflow保存hmap.buckets的溢出桶
	// oldoverflow保存hmap.oldbuckets的溢出桶
	// 这种间接引用使得可以将切片指针存储在hiter中
	overflow *[]*bmap
	oldoverflow *[]*bmap
	// nextOverflow持有指向空闲溢出桶的指针
	nextOverflow *bmap
}
```

**可以看出，每个bucket都是一个bmap**
bmap的结构体包含
+ tophash [8]uint8，存储键的哈希值的高8位
+ key [8]keytype , 存储键
+ value [8]valuetype , 存储值
+ overflow, 指向溢出桶的指针
key/value分别连续存储，而不是交替存储，是为了避免内存对齐填充

```go
// A bucket for a Go map.
type bmap struct {
// tophash存储本桶内每个键的哈希值高8位
// const bucketCnt untyped int = abi.MapBucketCount // 8
tophash [bucketCnt]uint8
// 随后按照顺序存放了bucketCnt个键和bucketCnt个值
// 将键和值分别连续存储(而非键/值交替存储)，虽然增加了代码复杂度，但避免了内存对齐填充(例如map[int64]int8需要填充的情况)
// 末尾附加一个溢出桶指针
}
```

**bmap并没有key、elem的字段，如何找到它们**
```go

// 数据偏移量(data offset)理论上应该等于bmap结构体大小，但必须满足正确的内存对齐要求
// 对于amd64p32架构，即使指针为32位，仍然需要保持64位对齐
// bmap结构体只有一个字段tophash [8]uint8
dataOffset = unsafe.Offsetof(struct {
	b bmap
	v int64
}{}.v)

// 通过偏移量
k = add(unsafe.Pointer(b), dataOffset)
e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
elem = add(k, bucketCnt*uintptr(t.KeySize))
```

## make初始化原理



```go
// makemap函数实现go中make(map[k]v,hint)的底层map创建逻辑
// 当编译器判定该map或首个bucket可在栈上分配时，返回值h 和/或 bucket可能非nil：
// 若h!=nil可直接在h中构建整个map结构
// 若h.buckets!=nil 则其所指bucket可作为map的首个存储桶使用

func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 计算需要分配的内存
	// hint表示哈希表的长度即bucket的个数，
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		hint = 0
	}
	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	// 设置hash seed
	h.hash0 = uint32(rand())
	// 计算容纳目标元素数量所需的B参数大小
	// 当hint<0时，overLoadFactor直接返回false，因为hint值小于最小bucket容量(bucketcnt)
	// 简单来说，就是计算第一次大于等于hint的2^B的B
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
	
	// 分配初始哈希表内存
	// 当B==0时，buckets字段会延迟分配(在mapassign阶段处理)
	// 注意：若hint值比较大，内存清零操作可能耗时比较长
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		// 溢出桶不为空，将其存储在h.extra.nextOverfloe中
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
	return h
}
```

**初始化流程**
+ 首先根据传入的参数hint(buckets的长度)，计算需要分配的内存
+ new一个hmap, 设置hash seed
+ 计算能够容纳目标元素数量的B(简单来说，就是计算第一次大于等于hint的2^B的B)
+ 如果h.B!=0，为其分配内存，有溢出桶将其存储在h.extra.nextOverflow中
	+ (B\==0：延迟分配，在mapassign阶段处理)
## map访问原理

**mapaccess1----v:=ma\[ke]**
```go
// mapaccess1返回指向h[key]的指针。本函数从不会返回nil
// 如果键不在map中，会返回该elem类型零值的引用。
// 重要提示：返回的指针可能会保持整个map存活，因此请不要长期持有该指针
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.Key.Size_)
	}
	if asanenabled && h != nil {
		asanread(key, t.Key.Size_)
	}
	// 如果hmap为nil，或者元素个数为0
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		// 返回类型零值
		return unsafe.Pointer(&zeroVal[0])
	}
	// 如果当前有协程对该map进行写
	// 由于map不支持并发读写，将fatal
	if h.flags&hashWriting != 0 {
		fatal("concurrent map read and map write")
	}
	// 获取key对应的哈希值
	hash := t.Hasher(key, uintptr(h.hash0))
	// 获取bucketMask:2^B-1
	m := bucketMask(h.B)
	// 获取key对应的bucket(bmap)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
	// oldbuckets只有在迁移时不为nil
	// 此时正在发生数据的迁移
	if c := h.oldbuckets; c != nil {
		// 此时进行的增量迁移
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			// 过去使用的存储桶的数量是现在的一半
			// 需要通过将掩码值右移一位计算
			m >>= 1
		}
		// 计算key在oldbucket中的位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
		// 如果当前正在访问的bucket没有被迁移，在oldbuckets中对应的bucket中寻找目标key
		if !evacuated(oldb) {
			b = oldb
		}
	}
	// 获取hash值的高8位，并对其进行矫正
	top := tophash(hash)
bucketloop:
	// 在b及其b的溢出桶中寻找key
	for ; b != nil; b = b.overflow(t) {
		// 在每个bmap中寻找(8个tophash\8个key\8个alue)
		for i := uintptr(0); i < bucketCnt; i++ {
			// 首先比较tophash
			if b.tophash[i] != top {
				// emptyRest: 当前cell是空的，更高索引以及溢出桶也是空的
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// 找到相等的tophash之后，获取对应位置的key并进行比较
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			// 获取key值
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 比较key值
			if t.Key.Equal(key, k) {
				// 获取key对应的value
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				// 找到key返回对应的value
				// 否则的话进入下一个循环，访问b桶的溢出桶
				return e
			}
		}
	}
	// 没有找到，返回对应类型零值的引用
	return unsafe.Pointer(&zeroVal[0])
}

```
**访问流程**
+ hmap为nil，或者元素个数为0，直接返回对应类型的零值
+ 通过hashWriting标识判断当前有协程正在执行写入操作，fatal
+ 获取key对应的hash值，找到key在buckets中对应的bmap(b)
+ 如果oldbuckets不为nil，说明当前正在进行数据的迁移，重新获取key在oldbuckets中的bmap(oldb)
+ 如果oldb不为nil，在oldb及其溢出桶中寻找对应的key;(否则在新的buckets中的b寻找)
	+ 首先比较tophash
	+ 然后比较key
+ 找到直接返回，找不到返回对应类型的零值


**mapaccess2----v,ok:=ma\[1]**
与mapacess1一样只是多了一个返回值，用来标记key是否在map中存在

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapaccess2)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.Key.Size_)
	}
	if asanenabled && h != nil {
		asanread(key, t.Key.Size_)
	}
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0]), false
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map read and map write")
	}
	hash := t.Hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop	
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.Key.Equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}
```


## 渐进式迁移
**hashGrow---扩容**

```go
func hashGrow(t *maptype, h *hmap) {
	// 如果已经达到负载因子阈值，则进行扩容
	// 说明存在过多溢出桶，进行等量扩容(使得元素存储更加紧凑)
	bigger := uint8(1)
	// 负载因子是6.5
	// 如果count>8 && count > 6.5 * (1<<h.B) 则是双倍扩容
	// 反之的话，就是等量扩容
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	// 将当前的buckets变为oldbuckets
	oldbuckets := h.buckets
	// 申请新的newbuckets
	// bigger==1就是双倍扩容
	// bigger==0就是等量扩容
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
	
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	// 重置hmap的一些字段值
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	// h.buckets设置为新的newbuckets
	h.buckets = newbuckets
	// 迁移的进度
	h.nevacuate = 0
	// 溢出桶的数量
	h.noverflow = 0
	
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		// 将原先的溢出桶移到oldoverflow
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}
	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
	// map数据的实际拷贝操作是通过growWork()和evacuate()函数渐进式完成的
}

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
// overLoadFactor判断当count个元素放入1<<B个桶时是否超过负载因子

// const loadFactorDen untyped int = 2
// const loadFactorNum untyped int = loadFactorDen * bucketCnt * 13 / 16 // 13
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets判断对于拥有1<<B个桶的map来说，溢出桶数量是否已经超过合理阈值
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// 阈值设置的平衡考量
	// 过低，引发不必要的扩容操作
	// 过高，频繁伸缩的map会残留大量未使用的内存
	// 过多的量化标准
	// 溢出桶数量接近常规桶数量时触发(noverflow ~ 1<<B)
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}
```

**hashGrow做了什么？**
+ 首先根据负载因子判断，是等量扩容还是双倍扩容
+ 将buckets移到oldbuckets，为buckets重新分配两倍大或者一倍大的空间
+ 将原先的溢出桶移动到oldoverflow
+ 设置nextOverflow，重要：设置迁移进度

**扩容的时机**
超过负载因子或者溢出桶数量太多
+ 元素个数超过8并且超过6.5*(1<<h.B)，双倍扩容
+ 溢出桶的个数接近于正常桶的数量


**growWrok---真正的迁移**
```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 确保清理出即将使用桶对应的旧桶空间
	// 迁移当前正在访问的bucket
	evacuate(t, h, bucket&h.oldbucketmask())  
	// evacuate one more oldbucket to make progress on growing
	// 如果正在扩容(空间已经扩容)
	if h.growing() {
		// 迁移nevacuate指向的bucket
		evacuate(t, h, h.nevacuate)
	}
}
```

**迁移的策略**
+ 迁移当前正在访问的bucket
+ 迁移迁移进度指向的bucket

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	// b是oldbuckets中正在迁移的桶
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
	// 返回的是oldbuckets的长度
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO:如果没有迭代器在使用旧桶，则复用溢出桶而非新建桶
		// xy变量存储了x和y(低位与高位)两个迁移目的地的信息
		// evacDst，存储迁移的目的地
		var xy [2]evacDst
		x := &xy[0]
		// x.b存储目的bmap
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.BucketSize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.KeySize))
		// 等量扩容迁移目的地是xy[0]
		// 双倍扩容迁移目的地是xy[1]
		if !h.sameSizeGrow() {
			// 只有在扩容的时候计算y指针
			// 否则GC可能会看到错误的指针
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.BucketSize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.KeySize))
		}
		// 开始迁移b及其溢出桶
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.KeySize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.KeySize)), add(e, uintptr(t.ValueSize)) {
				top := b.tophash[i]
				// top<=emptyOne
				// emptyRest or emptyOne
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				// 正常的top应该大于等于minTopHash
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.IndirectKey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				// 等量扩容的迁移方向就是x
				// 双倍扩容的迁移方向需要重新计算
				if !h.sameSizeGrow() {
					// 计算哈希值以便决定迁移方向(x or y)
					hash := t.Hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.ReflexiveKey() && !t.Key.Equal(k2, k2) {
						// 
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}
	  
				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				} 
				// 计算tophash的状态是evacuatedX or evacuatedY
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				// 迁移的目的地
				dst := &xy[useY] // evacuation destination
				// 迁移目的桶已经满，将数据迁移到溢出桶中
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.KeySize))
				}
				// 迁移
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.IndirectKey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.Key, dst.k, k) // copy elem
				}
				if t.IndirectElem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.Elem, dst.e, e)
				}
				// 迁移位置递增
				dst.i++
				// 这些更新可能会导致指针超过key或者elem数组的末尾
				// 这没有问题，因为我们在桶末尾设置了溢出指针
				// 防止指针越过桶的边界
				dst.k = add(dst.k, uintptr(t.KeySize))
				dst.e = add(dst.e, uintptr(t.ValueSize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
		// 解除溢出桶的链接并清空key/elem以辅助垃圾回收
		if h.flags&oldIterator == 0 && t.Bucket.PtrBytes != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.BucketSize))
			// 保留b.tophash的值，因为evacuation状态存储在那里
			ptr := add(b, dataOffset)
			n := uintptr(t.BucketSize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
	// 迁移进度递增
	if oldbucket == h.nevacuate {
		// 核心工作就是h.nevacuate++
		advanceEvacuationMark(h, t, newbit)
	}
}
```

**数据迁移的流程**
+ 首先计算oldbuckets中需要迁移的bmap(b)，如果b已经迁移过就退出
+ 提前设置好迁移的目的地
	+ x: buckets的前半段
	+ y:buckets的后半段
+ 开始迁移b以及其溢出桶
	+ 访问b的8个槽，获取每个位置的tophash，如果是emptyRest或者emptyOne无须迁移
	+ 计算key的hash值，根据该值判断迁移的目的地是x还是y
	+ 将oldbuckets对应位置的tophash、key、elem迁移到buckets中的对应位置
	+ 继续迁移下一个元素

**如何判断迁移工作完成**
```go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}

// 判断迁移工作完成的核心代码
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		h.oldbuckets = nil
		.........
	}
```

**迁移进度等于oldbuckets长度，迁移工作完成**

## map赋值操作原理
```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
// 类似于mapaccess函数,如果键不存在于map中会为其分配一个槽位
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// h==nil,panic
	if h == nil 
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.Key.Size_)
	}
	if asanenabled {
		asanread(key, t.Key.Size_)
	}
	// 如果有其它协程正在写，fatal
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	// 计算key的hash值
	hash := t.Hasher(key, uintptr(h.hash0))
	  
	// 在调用t.Hasher之后才设置hashWriting标识
	// 因为t.hasher可能会panic
	// 这种情况下实际上并没有执行写入操作
	h.flags ^= hashWriting
	// 创建hmap时，设置的buckets长度为0或者没有设置
	// 就没有为h.buckets分配内存，此时需要为其分配内存
	if h.buckets == nil {
		h.buckets = newobject(t.Bucket) // newarray(t.Bucket, 1)
	}

again:
	// 计算key所在的bmap
	bucket := hash & bucketMask(h.B)
	// 如果正在数据迁移，先进行迁移工作
	// 迁移当前访问的bucket以及nevacuate指向的bucket
	if h.growing() {
		growWork(t, h, bucket)
	}
	// 获取key对应的bmap
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	// 获取key的tophash
	top := tophash(hash)

	// 记录可能的插入位置
	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		// 依次遍历bmap中的8个slot
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				// 当前slot为emptyRest or emptyOne 并且 inserti为nil
				// 找到一个可能的插入位置
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				}
				// emptyRest:当前slot以及更高的索引位置和溢出桶都为empty
				// 退出循环执行插入操作
				if b.tophash[i] == emptyRest {
					break bucketloop
					// 这里break是退出bucketloop下的for 循环
				}
				// 由于待插入的key可能存在，需要继续向下查找
				continue
			}
			// b.tophash==top
			// 比较key是否相等
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			// key不相等寻找下一个位置
			if !t.Key.Equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			// key已经存在，更新对应的值
			if t.NeedKeyUpdate() {
				typedmemmove(t.Key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			// 结束流程
			goto done
		}
		// 获取b的溢出桶
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		// 下一轮，继续搜索b的溢出桶
		b = ovf
	}
	
	// 未能找到key，分配新的单元添加新的条目
	// 如果达到最大负载因子或者有太多溢出桶
	// 并且当前不在扩容过程中，开始扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		// 这里只是扩容空间，还没有真正的进行数据迁移
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	// 未能找到可能的插入位置
	if inserti == nil {
		// 当前桶及其溢出桶都已经满，需要分配一个新的桶
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.KeySize))
	}
		
	// store new key/elem at insert position
	if t.IndirectKey() {
		kmem := newobject(t.Key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.IndirectElem() {
		vmem := newobject(t.Elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.Key, insertk, key)
	*inserti = top
	// 元素个数加1
	h.count++

done:
	// 如果有其它协程正在写，fatal
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	// 写入操作完成，消除hashWriting标识
	h.flags &^= hashWriting
	// 设置对应位置的elem值
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

**赋值流程**
对于访问操作，需要优先考虑oldbuckets
对于赋值操作，直接操作buckets即可

+ 如果hmap为nil，panic(assignment to entry in nil map)
+ 有其它协程正在写，fatal(concurrent map writes)
+ 计算key的哈希值hash，设置写标识(hashWriting，由于计算hash的过程可能发生panic，故先计算hash后设置写标识)
+ 如果h.buckets为nil(make初始化时B=0，h.buckets就为nil)，为其分配相应的内存
+ 1. 获取key在buckets中对应的位置，如果此时正在迁移，先进行迁移的操作
+ 2. 获取key对应的bmap(b)以及key-hash的tophash(top)，依次遍历b的8个slot(在遍历的过程中寻找可能的插入位置)
+ 3. 如果当前位置的tophash不等于top
	+ 当前位置的tophash为emptyRest或者emptyOne，找到可能的插入位置
	+ 当前位置的tophash为emptyRest，直接退出循环，当前位置就是最终的插入位置
	+ 进入步骤5
+ 4. 当前位置的tophash等于top，继续比较位置上的key和给定的key是否相等
	+ 相等，对已经存在的key进行更新，进入done结束流程
	+ 不相等，继续按照上面的步骤搜寻b的下一个slot已经溢出桶
+ 5. 如果b及其溢出桶找不到key
	+ 首先判断新添一个k/v是否需要扩容，需要并且还没开始扩容，先进行扩容(hashGrow)并从步骤1重新开始
	+ 不需要扩容，直接进行写入操作
		+ 寻找到可能的插入位置，在该位置写入新k/v
		+ 新建一个溢出桶，写入k/v
		+ 元素个数增加
+ 6. done:结束流程
	+ 如果有其它协程正在写, fatal(concurrent map writes)
	+ 写入操作结束，消除写标识(hashWriting)
## map删除原理

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := abi.FuncPCABIInternal(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.Key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.Key.Size_)
	}
	if asanenabled && h != nil {
		asanread(key, t.Key.Size_)
	}
	// hmap为nil 或者 元素个数为0 直接退出
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return
	}
	// 如果有其它协程正在写，fatal
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	// 计算哈希值
	hash := t.Hasher(key, uintptr(h.hash0))
	// 在调用t.hasher后设置hashWriting标记
	// 因为t.hasher可能会panic
	// 这种情况下并没有执行写入(删除)操作
	h.flags ^= hashWriting
	// 计算key在buckets中的位置
	bucket := hash & bucketMask(h.B)
	// 如果正在迁移，先进行数据迁移工作
	if h.growing() {
		growWork(t, h, bucket)
	}
	// 获取key对应的bmap
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	bOrig := b
	// 获取key的tophash
	top := tophash(hash)
search:
	// 遍历b及其溢出桶
	for ; b != nil; b = b.overflow(t) {
		// 依次遍历bmap的8个slot
		for i := uintptr(0); i < bucketCnt; i++ {
			// tophash不相等
			if b.tophash[i] != top {
				// b.tophash为emptyRest,说明当前位置以及之后的位置都为empty
				// 直接退出循环
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			// tophash相等，继续比较key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			k2 := k
			if t.IndirectKey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			// key不相等，继续搜索下一个位置
			if !t.Key.Equal(key, k2) {
				continue
			}
			
			// key相等，进行clear工作
			// Only clear key if there are pointers in it.
			// 仅当键中包含指针时才清除键值
			if t.IndirectKey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.Key.PtrBytes != 0 {
				memclrHasPointers(k, t.Key.Size_)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			if t.IndirectElem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.Elem.PtrBytes != 0 {
				memclrHasPointers(e, t.Elem.Size_)
			} else {
				memclrNoHeapPointers(e, t.Elem.Size_)
			}
			// 将tophash的状态设置为emptyOne
			b.tophash[i] = emptyOne
			// 如果桶尾部现在处于一连串的emptyOne状态
			// 将这些状态改为emptyRest
			// 最好是将此逻辑抽成单独的函数
			// 但是目前for循环不支持内联优化
			
			// 如果当前slot的下一个位置不是emptyRest进入notLast流程
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			// 尽可能多的设置emptyRest状态
			// i的下一个位置是emptyRest,i是emptyOne，所以将i设置为emptyRest
			// 接着判断i的前一个位置
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
		   // 元素个数减1
			h.count--
			// 重置哈希种子以增加攻击者重复触发哈希碰撞的难度，详见问题25237。
			if h.count == 0 {
				h.hash0 = uint32(rand())
			}
			// 退出循环
			break search
		}
	}
	// 如果有其它协程正在写入，fatal
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	// 消除写标识
	h.flags &^= hashWriting
}
```

**删除的流程**
+ hmap为nil或者元素个数为0，直接退出
+ 判断写标识，如果有其它协程正在写，fatal
+ 计算哈希值，设置写标识，计算key在buckets中的位置
+ 如果正在数据迁移，先进行迁移工作
+ 获取key对应的bmap(b)，以及tophash(top)，开始search
+ 1. 遍历b及其溢出桶，遍历bmap中的8个slot
+ 2. 首先比较tophash值
	+ 不相等
		+ 如果tophash为emptyRest，退出循环
		+ 继续向下搜索
	+ 相等，继续比较key值
+ 3. 比较key值
	+ 相等，进行clear操作，将当前位置设置为emptyOne
	+ 不相等，继续搜索下一个位置
+ 4. clear之后，尽可能多的设置emptyRest状态
	+ 如果当前位置的下一个位置不是emptyRest，进入notLast
	+ 将当前位置设置为emptyRest，并继续向前判断
+ 5. notLast:
	+ 元素个数递减
	+ 重新设置hash seed
+ 6. 没有找到对应的key
	+ 通过写标识判断其它协程正在写，fatal
	+ 消除写标识

## 公共辅助函数

```go

// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
	// Masking the shift amount allows overflow checks to be elided.
	return uintptr(1) << (b & (goarch.PtrSize*8 - 1))
}


// growing reports whether h is growing. The growth may be to the same size or bigger.
// 判断h是否正在扩容
func (h *hmap) growing() bool {
	// oldbuckets不为nil，就是正在扩容
	return h.oldbuckets != nil
}

// 判断当前bucket(bmap)是否被迁移
// 返回true表示，已经被迁移
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}


// tophash calculates the tophash value for hash.
// tophash计算给定hash值的tophash值
func tophash(hash uintptr) uint8 {
	// 获取hash值的高8位
	top := uint8(hash >> (goarch.PtrSize*8 - 8))
	// 为了防止计算出来的top值与minTopHash混淆
	// 将top值加上minTopHash
	if top < minTopHash {
		top += minTopHash
	}
	return top
}


// evacDst is an evacuation destination.
// evacDst是一个迁移的目的地
type evacDst struct {
	b *bmap // current destination bucket
	i int // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

// isEmpty reports whether the given tophash array entry represents an empty bucket entry.
// isEmpty 函数用于判断给定的tophash数组条目是否表示一个空的桶条目
func isEmpty(x uint8) bool {
	return x <= emptyOne
}

```