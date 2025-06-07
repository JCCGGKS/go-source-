

# sync.WaitGroup


## 使用方式
先来看一个例子，开启100个协程执行加法操作，开启10个协程读取加法的和。

```go
var s int32

func main() {
	for i := 0; i < 100; i++ {
		go func() {
			s += 10
			time.Sleep(time.Second * 2)
		}()
	}
	for i := 0; i < 10; i++ {
		go func(index int) {
			fmt.Println("index=", index, "sum=", s)
		}(i)
	}
	fmt.Println("sum=", s)

	time.Sleep(time.Second * 5)

}
```

 - 在以上实示例中，有这样一段代码"time.Sleep(time.Second*5)"是为了防止主函数main提前返回，其余goroutine来不及执行。但是有这样一个问题，我们无法确定100个执行加法的协程和执行10个读取的协程什么时候会执行完成，如果设置的等待时间太长会导致主协程等待的时间太长，如果设置的等待的时间太短，会导致任务还没有执行完成就提前退出。
 - 为了解决上述的问题，引入了waitgroup。主要的作用就是用来等待其余协程全部执行完成之后再推出。

**为什么主协程执行完后就会退出，不会等待子协程**
可以参考视频："[Hello Goroutine的执行过程](https://www.bilibili.com/video/BV1hv411x7we/?p=16&spm_id_from=pageDriver&vd_source=f94b5d0224ae5feb4e351a885ce6bb75)"


上述的例子用waitgroup进行改写

```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(110)
	//100个goroutine用来执行写操作
	for i := 0; i < 100; i++ {
		go func() {
			defer wg.Done()
			s += 10
			time.Sleep(time.Second * 2)
		}()
	}
	//10个goroutine执行读操作
	for i := 0; i < 10; i++ {
		go func(index int) {
			defer wg.Done()
			fmt.Println("index=", index, "sum=", s)
		}(i)
	}
	fmt.Println("sum=", s)

	wg.Wait()
}
```

 - 声明一个sync.WaitGroup，然后通过Add方法设置计数器的值，需要等待多少数量的协程就设置成多少。
 - 在每个协程执行完毕时调用Done方法，让计数器减1，告诉Wait该协程已经执行完毕。
 - 最后调用Wait方法一直等待，直到计数器的值为0，也即是所有的协程都执行完毕。此时主协程可以退出。

	<font color='red'>sync.WaitGroup适合协调多个协程共同做一件事的场景，</font>比如下载一个文件，假设使用10个协程，每个协程下载文件的1/10大小，只有10个协程都下载好了整个文件才算下载好。

## 底层原理
WaitGroup是一个结构体，先来看一下它的定义

```go
/*
WaitGroup可以等待一些数量的协程完成相关的逻辑操作
主协程调用"Add"方法增加需要等待的正在运行的协程的数量；
每一个协程执行完自己的逻辑之后调用"Done"方法将需要等待的协程数量减少。与此同时"Wait"方法会阻塞直到需要等待的协程全部退出。
waitGroup在第一次使用之后禁止复制
在Wait阻塞之前Done的调用是并发安全的
*/
type WaitGroup struct {
//type noCopy struct{}
	noCopy noCopy 
	/*
	高32bit充当计数器的功能，记录需要追踪的协程数量。
	低32bit记录的是，因为wait操作而被阻塞的协程数量。
	*/
	state atomic.Uint64 // high 32 bits are counter, low 32 bits are waiter count.
	sema  uint32//用来同步Add中的读操作和Wait中的写操作
}

```
看"Add\Done\Wait"方法之前先看一下，有关信号量的两个操作

```go
/*
Semacquire等待直到s中存储的数值大于0，然后自动减少它
它是一个简单的睡眠原语
*/
func runtime_Semacquire(s *uint32)
/*
Semrelease自动增加s中存储的数值，并通知一个等待该信号量的协程；
是一个简单的唤醒原语
handoff为true，直接把count传递给第一个waiter
skipframes是跟踪过程中要忽略的帧数，从runtime_Semrelease调用者开始计数。
*/
func runtime_Semrelease(s *uint32, handoff bool, skipframes int)
```

### Add
先来看一看"Add"方法

```go
/*
Add方法向WaitGroup的计数器添加delta，有可能变为负值；
如果计数器变为0，所有被Wait阻塞的协程都将被释放
如果计数器变为负数，则会产生panic

*/
func (wg *WaitGroup) Add(delta int) {
	if race.Enabled {
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}
	//高32位是计数器的值，所以需要将delta加到高32bits上
	state := wg.state.Add(uint64(delta) << 32)
	//获取计数器的值
	v := int32(state >> 32)
	//获取因wait操作阻塞等待的协程数量
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
		/*
		v==int32(delta)表示这是第一次执行Add操作需要与Wait操作进行同步；
		*/
		race.Read(unsafe.Pointer(&wg.sema))
	}
	//计数器的值小于0
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	//第一次执行Add操作但是waiter不为0，说明Add操作和Wait操作正在并发执行
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	//执行成功，直接退出
	//v>0 && w==0表明第一次执行
	//v>0 && w>0 非第一次执行
	//v==0 && w==0追踪完成
	if v > 0 || w == 0 {
		return
	}
	//Add操作和Wait操作不能并发执行
	//Wait操作在计数器为0的情况下不会增加waiter,直接退出即可
	//如果state中存储的值与应该要存储的值不一致，也就是发生了Add和waitde 并发，会产生panic
	if wg.state.Load() != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	//走到这个分支表明计数器的值变为0，需要通过释放sema唤醒阻塞等待的Wait
	// Reset waiters count to 0.
	//v==0 && w>0 需要通过原语唤醒正在沉睡的waiter
	wg.state.Store(0)
	for ; w != 0; w-- {
	//释放信号量
		runtime_Semrelease(&wg.sema, false, 0)
	}
}
```

### Done
接下来看一看"Done"方法

```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
//将计数器的值减1
	wg.Add(-1)
}
```

### Wait

```go
// Wait blocks until the WaitGroup counter is zero.
//阻塞直到计数器值变为0
func (wg *WaitGroup) Wait() {
	if race.Enabled {
		race.Disable()
	}
	//一直循环检查直到计数器的值变为0
	for {
	//获取state字段的值
		state := wg.state.Load()
		//高32位是计数器，记录追踪的协程数量
		v := int32(state >> 32)
		//低32位是阻塞等待的协程数量
		w := uint32(state)
		//如果在执行wait操作的时候，v的值为0也就是没有正在追踪的协程，直接退出，无需等待。
		if v == 0 {
			// Counter is 0, no need to wait.
			// 计数器的值变为0，无需等待
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			//计数器是0直接退出无需等待
			return
		}
		//计数器的值不为0，说明正在追踪协程，调用wait的协程需要等待
		//增加因wait而阻塞的协程数量
		// Increment waiters count.
		//cas保证并发安全
		if wg.state.CompareAndSwap(state, state+1) {
			if race.Enabled && w == 0 {
			/*
			Wait必须与第一个Add进行同步。Wait中的写操作和Add中的读操作要同步；
			确保只有第一个等待者的写操作能执行成功，其余并发"wait"操作可以等待第一个操作完成后再继续
			*/
				race.Write(unsafe.Pointer(&wg.sema))
			}
			//等待sema中存储的数值大于0
			//这个操作会让wait操作一直阻塞直到Add操作释放信号量[也就是计数器的值变为0，等待的协程都退出]
			runtime_Semacquire(&wg.sema)
			if wg.state.Load() != 0 {
			//正要准备退出的时候，waitgroup又被reuse
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
			//退出阻塞
		}
	}
}

```
### 总结
对于Add操作

 - 原子操作将state高32bits计数器的值增加delta
 - 获取state中计数器的值为v，阻塞等待的协程数量为w
 - 如果v==delta则表明是第一次执行add操作，需要同步给wait
 - 如果 v<0,计数器的值小于0，产生panic
 - 如果v==delta但是w!=0说明Add操作和wait操作并发执行产生panic
 - v==0 && w>0说明计数器变为0，并且有被wait操作阻塞的协程，此时需要将state的值变为0，并以此释放信号量唤醒阻塞等待的waiter。
 -  !(v==0 && w>0) 也就是(v>0 || w\==0)，执行成功直接退出[既没有产生并发操作，也没有需要唤醒的waiter]
 
 对于wait操作
 
 - 获取state中计数器的值v，和阻塞等待的协程数量w
 - 如果v==0,无需等待直接退出
 - 否则表明正在追踪协程需要阻塞，用CAS操作将state的值加1表明此时有一个waiter，并阻塞等到Add操作释放信号量。
# sync.Once
## 存在的意义

 - 在实际的工作中，可能会遇到这样的需求：让代码只执行一次。哪怕是在高并发的情况下，比如创建一个单例。
 - sync.Once是个只执行一次“相关操作”的对象，它的作用是延迟初始化并且只初始化一次，可以用于初始化配置文件等。其实也可以使用init函数进行配置文件的初始化，但是init函数做不到“何时使用何时初始化”，init就好像“饿汉模式”，sync.Once就好像“懒汉模式”


## 使用方式
### 第一个例子，开启十个协程利用once运行同一个函数
```go
func main() {
	doOnce()
}
func doOnce() {
	var once sync.Once

	stop := make(chan bool)

	for i := 0; i < 10; i++ {
		go func() {

			once.Do(func() {
				fmt.Println("Do Once!!")
			})
			stop <- true
		}()

	}
	for i := 0; i < 10; i++ {
		<-stop
	}
}
```
运行结果
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/66d03bf20f903da53437cde12ae4b037.png)
另外一种写法为

```go
func main() {
	doOnce()
}
func doOnce() {

	var once sync.Once
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			once.Do(func() {
				fmt.Println("Do Once!!")
			})
		}()

	}
	wg.Wait()

}
```
### 第二个例子，懒汉单例获取配置文件

```go
// define the config
type Config struct {
	con map[string]int
}

var (
	//define the example of config
	config *Config
	once   sync.Once
)

func GetInstance() *Config {
	once.Do(func() {
		config = &Config{
			con: map[string]int{
				"v1": 1,
				"v2": 2,
			},
		}
	})
	return config
}
func main() {
	con := GetInstance()
	con1 := GetInstance()
	fmt.Println(con == con1)
}
```
运行结果
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1eaa9995c9d15882c07edbe1ebdf3be3.png)



## 底层原理

看一下sync.Once的结构组成

```go
type Once struct {
	/*
	done用来标识"相关操作是否已经被完成"，
	将它放在结构体的首位是为了实现更紧凑的指令
	*/
	done uint32
	m    Mutex
}
```

接下来看一下sync.Do方法

```go
/*
由于done标识的存在即使Do被调用多次，f也只会被调用一次。
如果f中调用Do会产生死锁。
如果在调用f的过程中产生panic,会认为这已经被执行过，再次调用也不会继续再次执行f，下面有例子演示
*/
func (o *Once) Do(f func()) {
/*
Note: Here is an incorrect implementation of Do:
if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	f()
}
这个实现看上去没有什么问题，逻辑上说得通；但是，如果同时有两个
goroutine进行cas的判断，cas的赢家会调用f()此时cas的输家不会立即返回
而是会等待cas的赢家执行完f之后才会返回，此时的性能是比较低的。而对于
sync.Do的实现，则是将逻辑分成了两部分，一部分是"fast-path"主要判断
done标识，如果已经执行过立即返回无需等待f的执行完成；另一部分是"slow-
path"进行f的执行
*/
	
	//fast-path
	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}
```

**slow-path**

```go
func (o *Once) doSlow(f func()) {
	//加锁
	o.m.Lock()
	defer o.m.Unlock()
	//双层检索
	if o.done == 0 {
		//修改done标识
		defer atomic.StoreUint32(&o.done, 1)
		//进行f的调用
		f()
	}
}
```


### 存在的问题
演示一下，执行f的过程中发生了panic
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e12d3618f72221730c914cf035b2050a.png)

 - 可以发现，如果f的执行过程中发生了panic就会中断f的执行，并且再次执行时也不会重新调用f。联系到实际的场景就是，如果在执行核心逻辑的过程中，某个步骤出现了问题就会导致，逻辑处理不成功，再次调用时也无法重新执行逻辑。
   
   
### 改进sync.Once解决问题
 - 产生问题的原因是，执行逻辑的途中产生错误，依然会设置done标识，再次重新执行的时候不会重新执行逻辑。
 - 解决的方式就是只有在成功执行逻辑不产生错误的情况下，才设置done标识

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a881268097da48bdae93c590264d31f1.png)

测试结果如下：
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d98ac14755ecd14db776728d45430a6d.png)
可以发现，f的执行过程中发生错误，下次调用仍会重新执行








# sync.Cond

 - sync.Cond从字面意义看是条件变量，具有阻塞和唤醒协程的功能，所以在满足一定条件的情况下唤醒协程。

## 使用方式

 - 以10个人赛跑为例。有一个裁判，裁判先等这10个人准备好，然后一声发令枪响，这10个人就可以跑了。

```go

func main() {
	race()
}
func race() {
	wg := sync.WaitGroup{}
	wg.Add(11) //10个参赛选手，1个裁判
	cond := sync.NewCond(&sync.Mutex{})
	for i := 0; i < 10; i++ {
		go func(index int) {
			defer wg.Done()
			fmt.Printf("%d号选手已经就位，等待枪响!\n", index)
			cond.L.Lock()
			defer cond.L.Unlock()
			cond.Wait()
			fmt.Printf("%d号选手听到枪响，开始running\n", index)
		}(i)
	}
	//等待所有选手就绪
	time.Sleep(time.Second * 2)

	go func() {
		defer wg.Done()
		fmt.Println("枪响")
		cond.Broadcast()
	}()
	wg.Wait()
}
```

运行结果

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4ed87e7d7f2ead0524f6b2598d407ec6.png)

 - 通过sync.NewCond函数生成一个*sync.Cond，用于阻塞和唤醒协程
 - 启动10协程模拟10个人，准备就位调用cond.Wait()方法阻塞当前协程等待发号施令，<font color='red'>调用sync.Cond()之前需要先加锁</font>。
 - time.Sleep用于等待所有人都进入wait阻塞状态，这样裁判才能发出口号。
 - 裁判准备完之后，调用cond.Broadcast()通知所有人开始跑步。

## 底层原理
首先看一下cond的底层结构
```go
/*
Cond实现一个条件变量，它是等待或宣布事件发生的goroutine的集合点。
每个Cond都有一个相关联的锁L(通常是*Mutex或*RWMutex)，在改变条件和调用Wait方法时必须持有它。
也即是说调用cond.Wait()方法之前必须先对cond相关联的锁执行加锁操作。

第一次使用后不得复制。在Go内存模型的术语中，Cond安排对Broadcast或Signal的调用在它解除阻塞的任何Wait调用之前“同步”。
对于许多简单的用例，用户最好使用通道而不是Cond (Broadcast对应于关闭一个通道，Signal对应于在一个通道上发送)。

有关同步替换的更多信息。其次，请参阅[Roberto Clapis关于高级并发模式的系列文章]以及[Bryan Mills关于并发模式的演讲]。
[Roberto Clapis的高级并发模式系列列]:https://blogtitle.github.io/categories/concurrency/
[Bryan Mills关于并发模式的演讲]:https://drive.google.com/file/d/1nPdvhB0PutEJzdCq5ms6UI58dp50fcAN/view
*/
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker //Locker is a interface

	notify  notifyList
	checker copyChecker
}
```
**NewCond方法**
根据传入的"Locker型变量"构造一个*Cond类型变量并返回
```go
// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```

**Wait方法**

```go
/*
Wait自动解锁c.L并挂起当前正在执行的goroutine。
在稍后恢复执行之后，Wait在返回之前锁定c.L。
与其他系统不同，Wait不能返回，除非被Broadcast或Signal唤醒。
因为在Wait等待时c.L没有被锁定，所以当Wait返回时，调用者通常不能假设条件为真。
*/
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	//暂时将与cond相关联的锁进行解锁
	c.L.Unlock()
	//等待直到被signal或者broadcast唤醒
	runtime_notifyListWait(&c.notify, t)
	//被唤醒之后再次加锁
	c.L.Lock()
}
```
**Signal方法**


```go
/*
信号唤醒一个等待cond的程序(如果有的话)。
Signal()不影响程序调度优先级;如果其他运行例程正在试图锁定c.L，它们可能会在“等待”运行例程之前被唤醒。
*/
func (c *Cond) Signal() {
	c.checker.check()
	//唤醒notifyList中的一个goroutinne
	runtime_notifyListNotifyOne(&c.notify)
}
```

**Broadcast**

```go
// It is allowed but not required for the caller to hold c.L
// during the call.
func (c *Cond) Broadcast() {
	c.checker.check()
	//唤醒所有的goroutine
	runtime_notifyListNotifyAll(&c.notify)
}
```

# 参考文章
[Go源码解析---sema.go](https://zhuanlan.zhihu.com/p/639599560)
