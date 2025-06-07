
不要通过共享内存通信，通过通信共享内存
`Don't communicate by sharing memory , share memory by communication`



# 基本用法

## 通道声明与初始化

+ 一个未初始化的通道会被置为nil
```go 
var ch chan int
fmt.Println(ch)
// <nil>
```

+ 通过make初始化
```go
var c = make(chan int) // 无缓冲通道
var c = make(chan int,10) // 有缓冲通道
```

## 通道写入数据

+ 对于无缓冲通道，能够写入数据的前提必须是有另一个协程在读取通道，是一个同步的过程
```go
func main() {
	a:=make(chan int)
	go func(){
		fmt.Println(<-a)
	}()	  
	a<-1
}
```

## 通道读取数据

+ 直接读取数据
```go
data := <-c
```

+ 第一个返回值表示读取的数据，第二个返回值表示通道是否关闭
```go
data,ok:=<-c
```

## 通道关闭

如果读取通道是一个循环操作，下例会出现大的问题---关闭通道不能终止循环，会收到无休止的零值序列

```go
func main() {
	ch := make(chan int)
	// ch:=make(chan int,1)
	close(ch)
	for {
		data, ok := <-ch
		fmt.Println(data, ok)
	}
}
// 0 false
// 0 false
// 0 false
...
// 0 false
```

+ 正确方法，用ok判断通道是否关闭
```go
func main() {
	ch := make(chan int)
	close(ch)
	for {
		data, ok := <-ch
		if ok{
			fmt.Println(data, ok)
		}else{
			fmt.Println("通道已经关闭\n")
			break
		}
	}
}
// 通道已经关闭
```

+ 或者使用for range循环，通道关闭之后会将剩余的数据取出，然后直接退出for 循环，不会读取零值

```go
func main() {
	ch := make(chan int)
	close(ch)
	for num:=range ch{
		fmt.Println(num)
	}
	fmt.Println("退出循环，通道关闭")
}
// 退出循环，通道关闭
```

+ 在实践中，不需要关心通道是否关闭，当通道没有被引用时会被GO的垃圾自动回收器回收
+ 关闭通道会触发所有等待读取的协程

## select
在实践中使用通道，通常与select结合。因为时常会出现多个通道多个协程进行通信的情况。
每个case语句必须对应通道的读写操作

+ 阻塞select
```go
select{
case <-ch1:
}
```

+ 非阻塞select
```go
select{
case <-ch1:

default:
}
```

select含有default分支就是非阻塞，不含有就是阻塞

### select随机选择机制

多个通道同时准备好读写操作，select的选择具有一定的随机性。
```go
func main() {
	ch := make(chan int, 1)
	ch<-2
	select {
	case <-ch:
		fmt.Println("数据已经准备好")
	case <-ch:
		fmt.Println("shujuyijingzhunbeihao")
	}
}
// 第一次运行结果
// shujuyijingzhunbeihao
// 第二次运行结果
// 数据已经准备好
```
### select堵塞与控制
select中没有任何准备好的通道，当前select所在的协程会陷入阻塞(有default分支的话，就不会阻塞)，直到有一个case准备好

**超时定时器实现非阻塞**
```go
func main() {
	ch := make(chan int, 1)
	select {
	case <-ch:
		fmt.Println("数据已经准备好")
	case <-ch:
		fmt.Println("shujuyijingzhunbeihao")
	case <-time.After(time.Second):
		fmt.Println("timeout")
	// default:
	// fmt.Println("default")
	}
}
// timeout
```

### 循环select

由于select每次选择分支都具有随机性，为了不执行一次就退出，使用select搭配for 循环

```go
for {
	select{
	}
}
```

### select 与 nil

当select语句的case对nil通道进行操作时，case分支将永远不会执行
**交替写入通道**

```go
func main() {
	a := make(chan int)
	b := make(chan int)
	go func(){
		for range 2 {
			select {
			case a <- 1:
				a = nil
			case b <- 2:
				b = nil
			}	
		}	
	}()
	fmt.Println(<-a)
	fmt.Println(<-b)
}
// 1 
// 2
```

# 底层原理
`runtime/chan.go`
## 源码给出的说明

本文件实现go语言的通道(channel)功能

不变式保证：
1. c.sendq和c.recvq至少有一个为空(等待发送队列 和 等待接收队列)
2. 唯一例外情况是：无缓冲通道中某个goroutine在select中同时阻塞发送和阻塞接收操作时，此时两个队列的长度仅受限于select语句的大小限制

对于带缓冲通道，还需满足：
1. 当缓冲区数量c.qcount>0时，接收队列c.recvq必须为空
2. 当缓冲区未满c.qcount < c.dataqsiz时，发送队列c.sendq必须为空
## hchan结构
channel是直接分配到堆上的，因为从设计理念上看是用于协程之间的通信，作用域和生命周期不会被限制在一个函数
**hchan**
```go
type hchan struct {
	qcount uint // 环形数组中元素的个数
	dataqsiz uint // 环形数组的大小
	buf unsafe.Pointer // 指向环形数组的指针
	elemsize uint16 // 环形数组中存储的元素类型大小
	closed uint32 //判断通道是否被关闭
	elemtype *_type // 环形数组中存储的元素类型
	sendx uint // 发送索引
	recvx uint // 接收索引
	recvq waitq // 接收等待队列
	sendq waitq // 发送等待队列

	// lock保护hchan中的所有字段，以及被此通道阻塞的sudogs中的若干字段
	// 持有锁时不要更改其它G的状态，特别是不要ready一个G，因为可能栈收缩产生死锁
	lock mutex
	// 锁用来保护读写关闭操作，实现并发安全
}
```

**waitq---尾插头取**
```go
type waitq struct {
	first *sudog // 等待队列的队首指针
	last *sudog // 等待队列的队尾指针
}
```

**sudog**
sudog可以看作是对阻塞挂起的goroutine的一个封装
```go
// sudog(伪协程)表示等待队列中的一个协程g
// sudog的存在是有必要的，因为g同步对象的关系是多对多的
// 1.一个g可能处于多个等待队列，一个g可能对应多个sudog
// 2.多个g可能同时在等待同一个同步对象，因此一个对象可能对应多个sudog
type sudog struct {
	// 下列字段由sudog阻塞所在的channel的hchan.lock保护
	// 对于涉及通道操作的sudog，shrinkstack功能依赖于此保护机制
	g *g
	next *sudog
	prev *sudog
	// 发送数据的地址，或者，接收接收数据的地址
	elem unsafe.Pointer // data element (may point to stack)

	................
	
	// isSelect表示goroutine正参与select操作
	// 因此必须通过CAS操作设置g.selectDone才能赢得唤醒竞争
	isSelect bool
	
	// success表示通道c的通信是否成功
	// 如果goroutine被唤醒是因为有值通过通道c传递，则为true
	// 如果通道因为被关闭而唤醒，则为false
	success bool
	
	waiters uint16
	parent *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c *hchan // channel
}
```

## 通道初始化

通过make初始化的channel是一个\*hchan类型(引用类型)

```go
func makechan(t *chantype, size int) *hchan {
	// 获取通道类型
	elem := t.Elem
	// compiler checks this but be safe.
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}
	
	// 计算需要分配的内存大小
	// elem.Size_ : 元素类型大小
	// size: 元素个数
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// 当buf中存储的元素不包含指针时，hchan结构体中也不包含GC感兴趣(需要跟踪)的指针
	// buf指向同一块内存分配区域，elemtype字段也是持久存在的
	// sudog对象由其所属线程持有引用，因此不会被垃圾回收
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		// 环形数组为空，即是无缓冲通道； element size为0，即是元素类型占用空间为0
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.PtrBytes == 0:
		// 元素不包含指针
		// 一次性分配hchan结构体和环形数组buf
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		// 计算c.buf的起始地址
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针，分为两次分配
		// 第一次分配hchan结构体
		// 第二次分配环形数组c.buf
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
	
	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c
}
```

**内存分配三步走**
+ 元素个数为0(无缓冲通道)或者元素类型占用空间为0
	+ 只分配hchan结构体
+ 不包含指针数据
	+ 一次性分配hchan结构体和环形数组
+ 包含指针数据
	+ 一次分配hchan结构体
	+ 一次分配环形数组(需要跟踪进行垃圾回收)
## 向通道发送数据

**辅助函数---full**
```go
// full函数判断向通道c发送数据是否会阻塞
func full(c *hchan) bool {
	// c.dataqsiz是只读字段(通道创建后不可修改)
	// 因此在通道操作的任何时刻读取该字段都是线程安全的
	if c.dataqsiz == 0 {
		// Assumes that a pointer read is relaxed-atomic.
		return c.recvq.first == nil
	}
	// Assumes that a uint read is relaxed-atomic.
	return c.qcount == c.dataqsiz
}
```
+ 对于无缓冲通道，写入的前提必须是，有另一个协程正在读
+ 对于有缓冲通道，写入的前提是缓冲区未满

```go
// 如果block参数不为nil，则在无法完成操作时，不会休眠，而是直接返回
// 当参与的通道被关闭时，sleep唤醒的g.param可能为nil，
// 最简单的处理方式是循环重试操作，此时可以检测到通道已经关闭
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		// block为false，直接返回
		if !block {
			return false
		}
		// 陷入阻塞，协程被挂起
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}
	
	if debugChan {
		print("chansend: chan=", c, "\n")	
	}
	
	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(chansend))
	}
	
	
	// full(c)：判断向通道c发送数据是否会阻塞；true:发送会阻塞
	// fastpath:在不获取锁的情况下检查非阻塞操作的失败条件
	if !block && c.closed == 0 && full(c) {
		return false
	}
	
	  
	
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	// 加锁操作
	lock(&c.lock)
	if c.closed != 0 {
		// 通道已经关闭，解锁，返回panic
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	// 获取读等待队列的第一个协程
	if sg := c.recvq.dequeue(); sg != nil {
		// 发现一个等待中的接收者，直接将数据发送给该接收者(绕过通道缓冲区，如果有的话)
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	
	  
	// 有缓冲且缓冲区未满
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		// 将数据发送到sendx的位置处
		// chanbuf:获取环形数组中sendx索引的地址
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		// 将ep的值拷贝到sendx
		typedmemmove(c.elemtype, qp, ep)
		// 计算下一个发送的位置
		c.sendx++
		// 环形数组：如果下一个发送的位置抵达数组末端，从0开始
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// 环形数组中元素个数加1
		c.qcount++
		// 解锁
		unlock(&c.lock)
		return true
	}
	// block为false直接返回
	if !block {
		unlock(&c.lock)
		return false
	}
	
	  
	
	// Block on the channel. Some receiver will complete our operation for us.
	// 在通道上阻塞
	// 获取当前协程g
	gp := getg()
	// 获取新的sudog结构
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	
	// 为当前阻塞的协程g绑定新的sudog结构mysg
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 将与当前协程绑定的sudog添加到sendq
	c.sendq.enqueue(mysg)
	
	
	// 向所有尝试收缩栈的代码发出信号：将要在channel上暂停
	// 在G的状态改变和设置gp.activeStackChans之间的窗口期是不安全的(不可进行栈收缩的)
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
	
	// 确保要发送的值在接收方完成拷贝前不会被GC回收。
	// sudog虽然有指向栈对象的指针，但不被视为栈追踪的根对象
	KeepAlive(ep)
	
	
	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	
	gp.waiting = nil
	gp.activeStackChans = false
	// success:true表示有值传递
	// false表示通道被关闭
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 通道因为关闭被唤醒
		panic(plainError("send on closed channel"))
	}
	return true
}
```

**发送流程**
前提是channel不为nil并且没有关闭
+ 通过fastpath判断(未获取锁)，发送是否会阻塞，核心在于full(c)
	+ 对于无缓冲通道，没有同步等待读取的协程，会阻塞
	+ 对于有缓冲通道，缓冲区已满，会阻塞
+ 首先获取recvq队头的receiver，将待发送的数据直接拷贝给该receiver
+ 如果缓冲区未满，将待发送的数据拷贝给c.sendx
+ 以上情况都不成立，该协程会被阻塞
	+ 创建新的sudog结构，与当前协程进行绑定
	+ 将sudog添加到c.sendq
+ 如果该阻塞的协程因为通道关闭被唤醒，此时会panic
	+ send on closed channel
## 从通道中读取数据

**辅助函数----empty(c)**
```go
// empty判断从通道c读取是否会发生阻塞(通道是否为空)
func empty(c *hchan) bool {
	// 无缓冲通道
	if c.dataqsiz == 0 {
		return atomic.Loadp(unsafe.Pointer(&c.sendq.first)) == nil
	}
	// 有缓冲通道
	return atomic.Loaduint(&c.qcount) == 0
}
```
**读取数据成功的前提条件**
+ 对于无缓冲通道，必须有另一个协程等待写入
+ 对于有缓冲通道，通道中的元素个数不为0

```go
// chanrecv函数从通道c接收数据并写入ep指针
// ep可以为nil,此时接收到的数据被丢弃
// 当block==false且没有可用元素，返回false,false
// 通道关闭，ep被清零，返回true,false
// 否则，将元素存入ep并返回true,true
// 非nil的ep必须指向堆内存或者调用者的栈空间
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.
	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}
		
	if c == nil {
		if !block {
			return
		}
		// 因为从nil chan接收数据而被阻塞
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
		throw("unreachable")
	}
	
	// fastpath:在未获取锁的情况下检查非阻塞操作是否失败
	// empty(c):true表示读取会被阻塞
	if !block && empty(c) {
		// 当检测到通道未就绪数据之后，需要进一步检查通道是否已经关闭
		if atomic.Load(&c.closed) == 0 {
			// 由于通道一旦关闭就无法重新开启，后续观察到的通道未关闭的状态
			// 即可推断出在首次检查的时刻通道同样未被关闭。此时的行为逻辑等同于
			// 在第一次检查时就观测到该状态，并据此返回接收操作无法继续执行的判断
			return
		}
		// 此时通道以不可逆的关闭。需要重新检查通道中是否存在待接收的数据
		// 这些数据可能出现在上述空状态检查和关闭检查之间的时间窗口。
		// 当与此类发送操作竞态时，此处同样需要保证顺序一致性
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}
	
	  
	
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	
	lock(&c.lock)
	// 通道已经关闭
	if c.closed != 0 {
		// 环形数组中元素个数为0
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			// 通道关闭，ep清零
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
		// The channel has been closed, but the channel's buffer have data.
	} else {
		// Just found waiting sender with not closed.
		if sg := c.sendq.dequeue(); sg != nil {
			// 发现sendq中的sender。
			// 若缓冲区大小为0，直接从发送方接收数据
			// 否则从环形数组头部接收数据，并将该发送方的值添加到环形数组的尾部
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}
	// 缓冲区中有可读的数据
	if c.qcount > 0 {
		// Receive directly from queue
		// 获取c.buf中c.recvx索引处的地址
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			// 将recvx处的值拷贝到ep
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		// c.recvx递增
		c.recvx++
		// recvx抵达队尾，从队头开始
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// 环形数组元素数递减
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	
	if !block {
		unlock(&c.lock)
		return false, false
	}
	
	  
	
	// no sender available: block on this channel.
	// 没有阻塞sender，该协程被阻塞
	// 获取当前协程gp
	gp := getg()
	// 新建一个sudog
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	// 将sudog与当前协程绑定
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)
	
	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// true:表示有值传递
	// false:表示通道被关闭
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

**接收数据流程**
前提是，通道是非nil
+ fastpath:在未获取锁的情况下检查非阻塞操作是否失败，核心是empty(c)，即检查一个通道是否是空的
	+ 通道未关闭且为空，直接返回
	+ 通道关闭且为空，返回true,false
+ 通道关闭，没有可读取的元素，返回true,false
+ 通道未关闭，获取sendq的sender
	+ 如果缓冲区中有元素，从缓冲区中读取，然后将sender的发送值拷贝到缓冲区
	+ 缓冲区中没有元素，直接从sender中读取
+ 缓冲区中有元素
	+ 直接从缓冲区中读取
+ 没有可用的sender
	+ 新建sudog结构，与当前协程绑定
	+ 将sudog加入到c.recvq
## 关闭通道

```go
func closechan(c *hchan) {
	// 通道为nil，报panic:close of nil channel
	if c == nil {
		panic(plainError("close of nil channel"))
	}
	
	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		// 通道已经关闭,报panic:close of closed channel
		panic(plainError("close of closed channel"))
	}
 
	
	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}
	
	  
	// 设置关闭标识为1
	c.closed = 1
	var glist gList
	// release all readers
	// 唤醒recvq中的所有recver
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		// 因通道关闭而被唤醒
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		// 把当前协程放回到全局协程队列
		glist.push(gp)
	}
	
	  
	
	// release all writers (they will panic)
	// 唤醒sendq中所有的sender，并且报panic:send closed channel
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		// 因通道关闭而被唤醒
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		// 被唤醒的协程放入到全局队列
		glist.push(gp)
	}
	unlock(&c.lock)
	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

**关闭流程**
+ 通道为nil，报panic: close of nil channel
+ 通道已经关闭(通过closed标识判断)，报panic: close of closed channel
+ 唤醒recvq中的所有recver，并将唤醒的g加入到glist
+ 唤醒sendq中的所有sender并报panic: send on closed channel。并将唤醒的g加入到glist


|       | nil                         | closed                         | 正常                        |
| ----- | --------------------------- | ------------------------------ | ------------------------- |
| send  | fata error                  | panic: send on closed channel  | 发送成功 or 阻塞                |
| recv  | fata error                  | 接收到类型的零值                       | 读取成功 or 阻塞                |
| close | panic: close of nil channel | panic: close of closed channel | 正常关闭，唤醒所有等待的recver和sender |

## select
`runtime/select.go`

**前置知识---切片**
```go
// 切片表达式
s[low:high:max]
// 长度：high-low
// 容量：max-low
// 这种方式创建的切片：可以限制切片的容量，新切片容量不会继承旧切片的容量


func main(){
	s:=[]int{1,2,3,4,5}
	newSlice:=s[:2:2]
	fmt.Println("Original slice:",s)
	fmt.Println("New Slice:",newSlice)
	fmt.Println("Length:",len(newSlice))
	fmt.Println("Capacity:",cap(newSlice))
}

// Original slice: [1 2 3 4 5]
// New Slice: [1 2]
// Length: 2
// Capacity: 2
```

### selectgo

**select中的每个case在运行时都是一个scase结构体**

```go
// select case描述符
// 编译器已知此结构
// 此处改动必须同步修改src/cmd/compile/internal/walk/select.go 中的 scasetype 结构
type scase struct {
	c *hchan // chan
	elem unsafe.Pointer // data element
}
```

**selectgo**

**select语句在运行时会调用selectgo函数**
+ 为了保持精简的栈空间使用量，scases的数量上限被设定为65536
+ case的类型有三种：select{recv/send/default}
+ nil通道会被忽略，因为nil通道永远处于阻塞状态

```go
// selectgo实现了select语句的核心功能
// cas0指向类型为[ncases]scase的数组，order0指向类型为[2*ncases]uint16的数组
// 其中ncases<=65536，这两个数组都存储在goroutine的栈上
// 无论在selectgo中是否有逃逸行为
// 在race detector的构建版本中，pc0指向类型为[ncases]uintptr的数组
// (同样位于栈上)；对于其它构建版本，该指针设置为nil

// 返回所选scase的索引，该索引对应select{recv/send/default}
// 如果选中的是recv，还会返回是否成功接收的布尔值标识
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	if debugSelect {
		print("select: cas0=", cas0, "\n")
	}

	// 为了保持精简的栈空间使用量，scases的数量上限被设定为65536
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
	// 总共的case数量为send+recv
	ncases := nsends + nrecvs
	// len=ncases , cap=ncases
	scases := cas1[:ncases:ncases]
	// pollorder代表乱序之后的scase序列
	pollorder := order1[:ncases:ncases]
	// 按照大小对通道地址进行排序
	lockorder := order1[ncases:][:ncases:ncases]



	var pcs []uintptr
	if raceenabled && pc0 != nil {
		pc1 := (*[1 << 16]uintptr)(unsafe.Pointer(pc0))
		pcs = pc1[:ncases:ncases]
	}
	casePC := func(casi int) uintptr {
		if pcs == nil {
			return 0
		}
		return pcs[casi]
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 生成随机排列顺序
	norder := 0
	for i := range scases {
		cas := &scases[i]
		// 从轮询顺序和加锁顺序中移除没有通道的case
		if cas.c == nil {
			cas.elem = nil // allow GC
			continue
		}
		j := cheaprandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]
	// 按照hchan地址排序获取加锁顺序
	for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

  
	if debugSelect {
		for i := 0; i+1 < len(lockorder); i++ {
			if scases[lockorder[i]].c.sortkey() > scases[lockorder[i+1]].c.sortkey() {
				print("i=", i, " x=", lockorder[i], " y=", lockorder[i+1], "\n")
				throw("select: broken sort")
			}
		}
	}

	// 按照lockorder的顺序锁住所有通道
	sellock(scases, lockorder)
	var (
		gp *g
		sg *sudog
		c *hchan
		k *scase
		sglist *sudog
		sgnext *sudog
		qp unsafe.Pointer
		nextp **sudog
	)

  

	// pass 1 - look for something already waiting
	// 第一轮排查----寻找已经就绪的通道
	
	var casi int
	var cas *scase
	var caseSuccess bool
	var caseReleaseTime int64 = -1
	var recvOK bool

	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		if casi >= nsends {
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}
		} else {
			if raceenabled {
				racereadpc(c.raceaddr(), casePC(casi), chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		}
	}

  

	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}

  
	
	// pass 2 - enqueue on all chans
	// 第二轮---在所有通道上排队等待
	
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		
		// 在分配元素和将sg加入gp.waiting队列之间
		// 禁止栈分裂，这样copystack才能正确找到它
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// 按照锁顺序构建等待队列
		*nextp = sg
		nextp = &sg.waitlink
		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}
	}
	// 等待被唤醒
	gp.param = nil
	// 在向任何试图缩减栈的人发出信号，表明即将在通道上休眠
	// 在此G的状态变更和设置gp.activeStackChans之间的窗口期是不安全进行栈缩减的
	gp.parkingOnChan.Store(true)
	gopark(selparkcommit, nil, waitReasonSelect, traceBlockSelect, 1)
	gp.activeStackChans = false	
	sellock(scases, lockorder)
	gp.selectDone.Store(0)
	sg = (*sudog)(gp.param)
	gp.param = nil
	
	// 第三轮处理---从未成功匹配的通道中出队
	// 否则这些请求会在空闲通道上堆积
	// 记录成功匹配的案例(如果有)
	// 按照锁定顺序将sudog单向连接起来
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			// sg已经被唤醒的G从队列中移除
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}
	
	if cas == nil {
		throw("selectgo: bad wakeup")
	}
	
	  
	
	c = cas.c
	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " send=", casi < nsends, "\n")
	}
	
	
	if casi < nsends {
		if !caseSuccess {
			goto sclose
		}
	} else {
		recvOK = caseSuccess
	}
	
	  
	
	if raceenabled {
		if casi < nsends {
			raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
		} else if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
	}
	if msanenabled {
		if casi < nsends {
			msanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			msanwrite(cas.elem, c.elemtype.Size_)
		}
	}
	if asanenabled {
		if casi < nsends {
			asanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			asanwrite(cas.elem, c.elemtype.Size_)
		}
	}
	selunlock(scases, lockorder)
	goto retc
	
bufrecv:
	// can receive from buffer
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
		racenotify(c, c.recvx, nil)
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.Size_)
	}
	if asanenabled && cas.elem != nil {
		asanwrite(cas.elem, c.elemtype.Size_)
	}
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc
	
bufsend:
	// can send to buffer
	if raceenabled {
		racenotify(c, c.sendx, nil)
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc
	
recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc
	  
rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc
	
send:
	// can send to a sleeping receiver (sg)
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc
	
retc:
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	return casi, recvOK
	
	
sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```


### 分步骤解析


selectgo函数中有两个关键的序列：pollorder和lockorder

**pollorder代表乱序后的case序列**


```go
	// 乱序，类似洗牌的算法
	// 生成随机排列顺序
	norder := 0
	for i := range scases {
		cas := &scases[i]
		// 从轮询顺序和加锁顺序中移除没有通道的case
		// nil通道会被移除
		if cas.c == nil {
			cas.elem = nil // allow GC
			continue
		}
		// cheaprandn is like cheaprand() % n but faster.
		// 生成[0,norder]之间的随机数
		j := cheaprandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]
```

**lockorder按照通道地址排序获取加锁顺序。规定加锁顺序是为了避免多个协程并发加锁时带来的死锁问题。**

```go
	// 大根堆排序
	for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		// 向上调整
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	// 向下调整
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
```

**按照lockorder的顺序锁住所有channel**
```go
func sellock(scases []scase, lockorder []uint16) {
	var c *hchan
	// 按照lockorder的顺序加锁
	for _, o := range lockorder {
		c0 := scases[o].c
		if c0 != c {
		// 去重，避免重复加锁
			c = c0
			lock(&c.lock)
		}
	}
}
```
**解锁**
```go
// 解锁的顺序和加锁的顺序相反
func selunlock(scases []scase, lockorder []uint16) {
	// 这里必须非常小心，在释放最后一把锁后就不能再接触sel变量
	// 因为sel可能在最后一次解锁后立即释放
	// 考虑以下场景：
	// 1.第一个M在runtine.selectgo()中调用runtime.park()并传入sel
	// 2.一旦runtime.park()释放了最后一把锁，另一个M使调用select的G再次可运行并调度执行
	// 3.当G在另一个M上运行时，会锁住所有锁并释放sel
	// 4.此时如果第一个M再访问sel，就会操作已释放的内存
	for i := len(lockorder) - 1; i >= 0; i-- {
		c := scases[lockorder[i]].c
		if i > 0 && c == scases[lockorder[i-1]].c {
			continue // will unlock it on the next iteration
		}
		unlock(&c.lock)
	}
}
```

**第一轮排查，寻找已经就绪的通道**
+ 由于pollorder是已经打乱顺序的序列，遍历的过程就是随机的过程
+ 如果当前case是recv:从通道中获取数据
	+ 获取正在等待写的sender，进入recv分支
	+ 缓冲区中有数据，进入bufrecv分支
	+ 通道已经关闭，进入rclose分支
+ 如果当前case是send:向通道中写入数据
	+ 通道已经关闭，进入sclose分支
	+ 获取正在等待读的recv，进入send分支
	+ 缓冲区未满，进入bufsend分支
+ 有default:非阻塞，进入retc分支


```go
	// pass 1 - look for something already waiting
	// 第一轮排查----寻找已经就绪的通道
	
	var casi int
	var cas *scase
	var caseSuccess bool
	var caseReleaseTime int64 = -1
	var recvOK bool
	// pollorder是打乱顺序的序列
	// 随机选择一个就绪的channel
	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		// case recv
		if casi >= nsends {
			sg = c.sendq.dequeue()
			// 获取等待写的sender
			if sg != nil {
				goto recv
			}
			// 没有等待写的sender，缓冲区中有数据
			if c.qcount > 0 {
				goto bufrecv
			}
			// 获取不到数据，通道关闭
			if c.closed != 0 {
				goto rclose
			}
			// case send
		} else {
			if raceenabled {
				racereadpc(c.raceaddr(), casePC(casi), chansendpc)
			}
			// 通道已经关闭
			if c.closed != 0 {
				goto sclose
			}
			// 获取等待读的recver
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			// 没有等待读的recver，缓冲区未满
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		}
	}
	
	// 含有default分支---非阻塞
	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}
```

**第二轮排查----在所有通道上排队等待**
```go
	// pass 2 - enqueue on all chans
	// 第二轮---在所有通道上排队等待

	// 获取当前协程
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	// lockorder:按照通道的地址进行排序
	// 按照lockorder的顺序进行排队等待
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		// 获取sudog结构与当前协程gp进行绑定
		sg := acquireSudog()
		sg.g = gp
		// 参与了select
		sg.isSelect = true
		
		// 在分配元素和将sg加入gp.waiting队列之间
		// 禁止栈分裂，这样copystack才能正确找到它
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// 按照锁顺序构建等待队列
		*nextp = sg
		nextp = &sg.waitlink
		// 将sudog塞入到相应的等待队列
		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}
	}
	// 等待被唤醒
	gp.param = nil
	// 在向任何试图缩减栈的人发出信号，表明即将在通道上休眠
	// 在此G的状态变更和设置gp.activeStackChans之间的窗口期是不安全进行栈缩减的
	gp.parkingOnChan.Store(true)
	gopark(selparkcommit, nil, waitReasonSelect, traceBlockSelect, 1)
	gp.activeStackChans = false	
	sellock(scases, lockorder)
	gp.selectDone.Store(0)
	sg = (*sudog)(gp.param)
	gp.param = nil
```

**第三阶段排查---将与协程绑定的sudog依次从通道队列中移除**
```go
	// 第三轮处理---从未成功匹配的通道中出队
	// 否则这些请求会在空闲通道上堆积
	// 记录成功匹配的案例(如果有)
	// 按照锁定顺序将sudog单向连接起来
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	// 在从gp.waiting断开链接前清除所有元素
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			// sg已经被唤醒的G从队列中移除
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			c = k.c
			// 将sudog从相应队列中移除
			// sglist记录了已经被移除的sudog
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}
	// cas==nil,没有一个通道被唤醒
	if cas == nil {
		throw("selectgo: bad wakeup")
	}
	
	c = cas.c
	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " send=", casi < nsends, "\n")
	}
	
	if casi < nsends {
		if !caseSuccess {
			goto sclose
		}
	} else {
		recvOK = caseSuccess
	}
	
	  
	
	if raceenabled {
		if casi < nsends {
			raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
		} else if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
	}
	if msanenabled {
		if casi < nsends {
			msanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			msanwrite(cas.elem, c.elemtype.Size_)
		}
	}
	if asanenabled {
		if casi < nsends {
			asanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			asanwrite(cas.elem, c.elemtype.Size_)
		}
	}
	selunlock(scases, lockorder)
	goto retc
```


**send分支**

```go
send:
	// can send to a sleeping receiver (sg)
	// 有正在等待读的receiver，将阻塞sender的数据直接拷贝给该receiver
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc
```
**bufsend分支**
```go
bufsend:
	// can send to buffer
	// 缓冲区有空闲位置，将数据发送到缓冲区
	if raceenabled {
		racenotify(c, c.sendx, nil)
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	// 拷贝数据
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	// sendx递增
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	// 元素个数递增
	c.qcount++
	selunlock(scases, lockorder)
	goto retc
```

**sclose**

```go
sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```

**recv分支**

```go
recv:
	// can receive from sleeping sender (sg)
	// 从正在写的协程接收数据
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	// recvOK为true表明收到数据
	recvOK = true
	goto retc
```
**bufrecv分支**
```go
bufrecv:
	// can receive from buffer
	// 从缓冲区中读取数据
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
		racenotify(c, c.recvx, nil)
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.Size_)
	}
	if asanenabled && cas.elem != nil {
		asanwrite(cas.elem, c.elemtype.Size_)
	}
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	// recvx递增
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	// 元素个数递减
	c.qcount--
	selunlock(scases, lockorder)
	goto retc
```
**rclose**

```go
rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	// recvOK为false，表明没有接收到值
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc
```

**retc分支**
```go
retc:
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	// 返回返回值，退出程序
	// casi：随机选中的case
	// recvOK:是否接受到值
	return casi, recvOK
```



