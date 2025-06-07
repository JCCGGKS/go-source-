# Mutex

## 源码说明
Mutex是一种互斥锁
Mutex的零值是一个未锁定的互斥体
Mutex在首次使用后不得复制
根据Go内存模型的术语：
+ 对于任意n\<m，第n次Unlock调用先于同步于(happens before)第m次Lock调用
+ 成功的TryLock调用等效于Lock调用
+ 失败的TryLock调用不会建立任何先于同步关系

## Mutex结构

Locker是一个接口，Mutex实现了这个接口。

```go
// A Locker represents an object that can be locked and unlocked.
// Locker代表一个可被锁定和解锁的对象
type Locker interface {
	Lock()
	Unlock()
}
```

```go
type Mutex struct {
	state int32 // 锁的标识
	sema uint32 // 信号量
}
```

```go
const (
	// mutex is locked
	mutexLocked = 1 << iota // 1
	mutexWoken              // 2
	mutexStarving           // 4
	mutexWaiterShift = iota // 3
	// 互斥锁公平性
	// 互斥锁(Mutex)有两种运行模式：正常模式和饥饿模式
	// 在正常模式下，等待者按照先进先出(FIFO)的顺序排队
	// 但被唤醒的等待者不会直接获得锁，而是要与新到达的goroutine竞争所有权
	// 新到达的goroutine有优势---它们已经在CPU上运行且数量可能很多，
	// 因此被唤醒的等待者很可能会竞争失败。这种情况下，会重新排到等待队列的头部。
	// 如果等待者超过1ms未能获取锁，互斥锁会切换到饥饿模式
	// 在饥饿模式下，锁的所有权会直接从解锁的goroutine移交给队列头部的等待者
	// 新到达的goroutine即使发现锁处于未锁定状态，也不会尝试获取锁，更不会自旋
	// 而是直接把自己加入到等待队列的尾部
	// 当等待者获得锁的所有权时，如果发现：
	// 1.自己是队列中最后一个等待者，或者
	// 2. 等待时间不足1ms
	// 就会将互斥锁切换回正常模式
	// 正常模式性能显著更好，因为即使存在阻塞的等待者，goroutine也可以多次连续多次获取锁
	// 饥饿模式对于防止尾部延迟的极端情况非常重要
	starvationThresholdNs = 1e6
	// 1ms=1e6ns
)
```



**Mutex含有的两个字段**
+ state 锁的标识
	+ `mutexLocked` (第0位): 1表示锁已被持有
	-  `mutexWoken` (第1位): 1表示有goroutine被唤醒
	-  `mutexStarving` (第2位): 1表示锁处于饥饿模式
	-  其余位: 表示等待锁的goroutine数量
+ sema 信号量
**锁的两种模式**
+ 正常模式
+ 饥饿模式
互斥锁公平性：
互斥锁(Mutex)有两种运行模式：正常模式和饥饿模式
在正常模式下，等待者按照先进先出(FIFO)的顺序排队
但被唤醒的等待者不会直接获得锁，而是要与新到达的goroutine竞争所有权
新到达的goroutine有优势---它们已经在CPU上运行且数量可能很多，
因此被唤醒的等待者很可能会竞争失败。这种情况下，会重新排到等待队列的头部。

如果等待者超过1ms未能获取锁，互斥锁会切换到饥饿模式
在饥饿模式下，锁的所有权会直接从解锁的goroutine移交给队列头部的等待者
新到达的goroutine即使发现锁处于未锁定状态，也不会尝试获取锁，更不会自旋
而是直接把自己加入到等待队列的尾部
当等待者获得锁的所有权时，如果发现：
1. 自己是队列中最后一个等待者，或者
2. 等待时间不足1ms
就会将互斥锁切换回正常模式
正常模式性能显著更好，因为即使存在阻塞的等待者，goroutine也可以多次连续多次获取锁
饥饿模式对于防止尾部延迟的极端情况非常重要



```go

```
## Lock

```go
// Lock方法用于锁定互斥锁m
// 如果该锁已经被占用，调用的goroutine将被阻塞，直到互斥锁变为可用状态
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	// 快路径：抓住未锁定的锁
	// 原子操作快速获取锁，得到返回，得不到进入slow path
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64
	// 标识是否处于饥饿模式
	starving := false
	// 标识是锁是否被唤醒
	awoke := false
	// 记录自旋的次数
	iter := 0
	// 保存旧的锁状态,m.state
	old := m.state
	// 循环获取锁
	for {
		// 当前锁的状态是锁定状态，并且可以进行自旋操作
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 主动自旋是合理的
			// 尝试设置mutexWoken标志位通知Unlock操作，从而避免唤醒其它被阻塞的goroutine
			// !awoke:唤醒标识为false
			// old&mutexWoken==0:之前未被唤醒过
			// old>>mutexWaiterShift!=0:阻塞等待的goroutine数量不为0
			// atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken):原子设置mutexWoken
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			// 自旋获取锁
			runtime_doSpin()
			// 自旋次数增加
			iter++
			old = m.state
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		// 不要尝试获取处于饥饿模式的互斥锁，新到达的goroutine必须排队等待。
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		// 锁已经被锁定或者处于饥饿模式
		// 锁等待协程数增加
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		
		// 当前goroutine将互斥锁切换至饥饿模式
		// 但如果互斥锁当前处于解锁状态，则不进行切换
		// Unlock操作预期饥饿模式的互斥锁应有等待者
		// 而当前情况并不符合条件
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// 该goroutine已从休眠中被唤醒
			// 因此无论哪种情况都需要重置标志位
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// 消除mutexWoken标识
			new &^= mutexWoken
		}
		// 原子替换当前的状态为new
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			// 若之前已经在等待，则插入队列前端
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			// 等待时间超过1ms
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// 如果该goroutine被唤醒且处于饥饿模式
				// 所有权虽然移交给我们，但互斥锁处于某种不一致状态
				// mutexLocked未设置，仍被记为等待者，需要修正此状态
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 获取锁的时间不超过1ms，或者，锁等待队列只有一个等待者，消除饥饿模式
				if !starving || old>>mutexWaiterShift == 1 {
					// 退出饥饿模式，必须在此处执行该操作并考虑等待时间
					// 饥饿模式效率极低，两个goroutine一旦将互斥锁切换为饥饿模式
					// 就可能无限循环地同步运行
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```


**加锁流程**
+ fast path
	+ 原子操作加锁(`atomic.CompareAndSwapInt32`)，加锁成功直接返回，不成功进入slow path
+ slow path
	+ 如果锁当前处于锁定状态`mutexLocked`并且可以进行自旋
		+ 首先尝试设置`mutexWoken`标识，防止唤醒其它协程
		+ 自旋，递增自旋次数
	+ 如果锁不处于饥饿模式`mutexStarving`，加锁成功，设置`mutexLocked`
	+ 如果锁为`mutexLocked`或者`mutexStarving`，加锁失败，锁等待协程数增加
	+ 如果饥饿标识`starving`为真并且处于`mutexLocked`状态，进入饥饿模式
	+ 如果唤醒标识`awoke`为真，重置标识消除`mutexWoken`
	+ 如果之前等待过，将当前协程加入到等待队列首部，否则加入尾部(获取信号量)
	+ 如果等待时间超过1ms，设置饥饿标识starving
	+ 等待时间不超过1ms或者锁等待者数量为1，切换回正常模式
	+ 设置唤醒标识`awoke`
**自旋的条件**
+ 
## Unlock

```go
 // Unlock 解锁 m
 // 如果调用Unlock时m未被锁定，fatal
 // 已经锁定的mutex并不与特定的goroutine绑定
 // 允许一个goroutine锁定mutex之后由另一个goroutine解锁
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}
	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// 将慢速路径单独提取出来以允许内联优化快速路径
		// 为了在跟踪时隐藏unlockSlow方法，当跟踪GoUnblock时会额外跳过一个堆栈帧
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// 如果已经没有等待者，或者已经有协程被唤醒/抢占了锁，则无需唤醒其它协程
			// 在饥饿模式下，锁的所有权会直接从解锁协程移交给下一个等待者
			// 我们并不在这个移交链中，因为之前解锁互斥锁时并未观察到mutexStarving标志
			// 所以这里直接退出即可
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			// 抢占唤醒其它协程的权利
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// 饥饿模式: 将互斥锁的所有权移交给下一个等待者，
		// 并主动让出当前时间片，使得下一个等待者可以立即开始运行。
		// 注意：此时尚未设置 mutexLocked 标志，等待者被唤醒后会自行设置该标志。
		// 但如果 mutexStarving 标志已设置，互斥锁仍被视为锁定状态，
		// 因此新到达的 goroutine 将无法获取该锁。
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```

**解锁的流程**
+ fast path
	+ 原子操作，丢弃lock bit，丢弃之后m.state不为0，进入slow path
+ slow path
	+ 如果解锁之前未上锁，fatal
	+ 处于饥饿模式
		+ 直接唤醒锁等待队列首部的等待者(释放信号量)
	+ 未处于饥饿模式
		+ 如果没有锁等待者或者有其它协程抢占了锁，直接返回
		+ 唤醒其它等待协程(直到锁重新被mutexLocked or mutexStarving or mutexWoken)，释放信号量
观察代码可以发现对于函数`runtime_Semrelease(&m.sema, true, 1)`和`runtime_SemacquireMutex(&m.sema, queueLifo, 1)`
第二个参数如果为true，表明是LIFO，为false表示FIFO
# RWMutex

## 源码说明

运行时(runtime)包中的rwmutex.go文件包含了本文件的修改副本。
如果在此处进行任何更改，请考虑是否也需要在那边做相应修改。

RWMutex是一个读写互斥锁。
该锁可以被任意数量的读者或单个写者持有。
RWMutex的零值是一个未锁定的互斥体。

RWMutex在第一次使用后不得复制。

当一个或多个读者已持有锁时，如果有goroutine调用Lock，
那么并发的RLock调用将会阻塞，直到写者获取（并释放）该锁，
这确保了锁最终能被写者获得。
请注意，这种方式禁止了递归的读锁定。

按照Go内存模型的术语表述：
对于任意n < m，第n次Unlock调用"同步先于"第m次Lock调用，
这与Mutex的行为一致。
对于每一次RLock调用，都存在一个n使得：
第n次Unlock调用"同步先于"该次RLock调用，
而相应的RUnlock调用"同步先于"第n+1次Lock调用。

## RWMutex结构
```go
type RWMutex struct {
	w Mutex // 
	writerSem uint32 // semaphore for writers to wait for completing readers
	readerSem uint32 // semaphore for readers to wait for completing writers
	readerCount atomic.Int32 // number of pending readers
	readerWait atomic.Int32 // number of departing readers
}
const rwmutexMaxReaders = 1 << 30
```

# RLock
```go

// 通过以下方式向竞态检测器指示happens-before关系：

// - Unlock -> Lock：通过readerSem信号量
// - Unlock -> RLock：通过readerSem信号量 
// - RUnlock -> Lock：通过writerSem信号量

// 下文中的方法会临时禁用对竞态同步事件的处理，
// 以便为竞态检测器提供上述更精确的内存模型。

// 例如，RLock中的atomic.AddInt32不应该表现出acquire-release语义，
// 否则会错误地同步竞态的读操作，从而可能导致漏检数据竞争。

// RLock对rw进行加读锁操作。

// 该方法不应用于递归读锁定；一个被阻塞的Lock调用会阻止新的读锁获取，
// 详见RWMutex类型的文档说明。
func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// readerCount递增
	// 递增之后小于0，说明
	if rw.readerCount.Add(1) < 0 {
		// A writer is pending, wait for it.
		// 有写入操作正在执行，请等待其完成
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}
```
**RLock流程**
通过观察代码`Lock`可以发现，当有写者执行`Lock`操作时，会执行`rw.readerCount.Add(-rwmutexMaxReaders)`
+ 递增readerCount
+ 如果readerCount小于0，说明有写者正在写
+ 获取读信号量
## RUnlock
```go
// RUnlock 撤销一次 RLock 调用； 
// 它不会影响其他同时进行的读操作。
// 如果在进入 RUnlock 时 rw 未被加读锁， 
// 则是运行时错误。
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	// 递减readerCount
	// 递减之后小于0，进入slow path
	if r := rw.readerCount.Add(-1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}
	if race.Enabled {
		race.Enable()
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		fatal("sync: RUnlock of unlocked RWMutex")
	}
	// A writer is pending.
	if rw.readerWait.Add(-1) == 0 {
		// The last reader unblocks the writer.
		// 所有读者退出，释放写信号量
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```
**RUnlock流程**
+ 递减readerCount，小于0，进入slow path
+ 递减readerWait，如果等于0，释放写信号量

## Lock
```go
// Lock 为写操作锁定 rw。 
// 如果该锁已被加读锁或写锁， 
// Lock 将阻塞直到锁可用。
func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	// 向读者宣告有等待的写者
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	// 如果r不为0，有读者正在读，递增需要等待的读者数
	if r != 0 && rw.readerWait.Add(r) != 0 {
		// 获取写信号量
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}
	
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
```
**Lock流程**
+ 调用mutex.lock进行加锁
+ 递减readerCount(-rwmutexMaxReaders)，向读者宣告有写者正在写入
+ 如果此时有读者正在读，需要递增readerWait，获取写信号量
通过以上的流程，可以发现读写锁是以读优先的
## Unlock
```go
// Unlock 解除 rw 的写锁定。如果在进入 Unlock 时 
// rw 未被加写锁，则是运行时错误。 
// 与 Mutex 类似，已锁定的 RWMutex 并不与特定的 
// goroutine 相关联。某个 goroutine 可以 RLock (Lock)  
// 一个 RWMutex，然后安排其他 goroutine 执行 RUnlock (Unlock)。
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// Announce to readers there is no active writer.
	// 向读者宣告没有活跃的写者
	r := rw.readerCount.Add(rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		fatal("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	// 唤醒所有等待的读者。如果有的话。
	for i := 0; i < int(r); i++ {
		// 释放读信号量
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
```
**Unlock流程**
+ 递增readerCount(rwmutexMaxReaders)，向读者宣告写者退出
+ 如果递增之后readerCount大于等于rwmutexMaxReaders，fatal(sync: Unlock of unlocked RWMutex)
+ 唤醒所有读者，释放读信号量