
# 底层原理
`sync/map.go`

## 源码给出的说明
Map类型类似于Go的map\[any]any，但支持多goroutine的并发安全访问，无需额外的锁或同步机制。
其加载(Load)、存储(Store)、删除(Delete)操作均摊时间复杂度为O(1)

该类型为特殊场景设计。大多数代码仍应使用原生go map配合显式锁，以获得更好的类型安全性，并更易于维护map内容的其它不变性条件。

Map类型针对两种典型场景优化
+ 键值对仅写入一次但多次读取(如，只增缓存)
+ 多goroutine读写操作涉及不相交的键集合
这两种场景下，相比原生map配合Mutex/RWMutex的方案，能显著降低锁竞争

零值Map即为空且立即可用。Map初始化后禁止复制

根据go内存模型术语，Map确保写操作同步于能观察到该写操作效果的所有读操作
操作类型定义如下：
+ 读操作：Load、LoadAndDelete、LoadOrStore、Swap、CompareAndSwap、CompareAndDelete
+ 写操作：Delete、LoadAndDelete、Store、Swap
+ LoadOrStore返回loaded=false时为写操作
+ CompareAndSwap返回swapped=true时为写操作
+ CompareAndDelete返回deleted=true时为写操作



## 底层结构

```go
type Map struct {
	mu Mutex
	// read字段包含了map中那些可以安全并发访问的部分内容(无论是否持有mu锁)
	// read字段本身总是可以安全加载，但更新时必须持有mu锁
	// read中的条目可以在不持有mu锁的情况下并发更新，但是若要更新一个已经被擦除(expunged)的条目
	// 需要将该条目复制到dirty map中，并在持有mu锁的情况下取消擦除(unexpunged)
	read atomic.Pointer[readOnly]

	// dirty字段保存了map中需要持有mu锁才能访问的部分内容
	// 为了确保dirty map可以快速提升为read map
	// 还包含了read map中所有未被擦除(unexpunged)的非过期条目
	// 被擦除(expunged)的条目不会存储在dity map中
	// 如果要向clean map中的一个被擦除的条目存储新值，必须在持有mu锁的情况下，取消擦除该条目并将其添加到dirty map 中
	// 如果dirty map为nil，map的下次写入操作会通过浅拷贝clean map(排除过期条目)来初始化它
	dirty map[any]*entry
	// misses计数器记录自read map上次更新后，需要加锁才能判断key是否存在数据访问次数
	// 当misses达到足够大(足以覆盖复制dirty map的成本时)，dirty map会被提升为read map(保持未修改状态)
	// 下次map存储操作会创建新的dirty副本
	misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
// readOnly是一个不可变(immutable)结构体，以原子方式(atomically)存储在Map.read字段中
type readOnly struct {
	m map[any]*entry
	amended bool // true if the dirty map contains some key not in m.
	// 当dirty map中包含了m中没有的key时返回true
}


// An entry is a slot in the map corresponding to a particular key.
// entry是map中对应于特定键的存储槽位
type entry struct {
	// p指向条目存储的interface{}值
	// 如果p==nil，表示条目已经被删除，此时m.dirty==nil或
	// m.dirty[key]等于当前条目e
	// 如果p==expunged，表示条目已经被删除，m.dirty!=nil
	// 但该条目不存在m.dirty中
	// 其它情况下，条目有效且被记录在m.read.m[key]中，
	// 如果m.dirty!=nil,也会记录在m.dirty[key]中
	// 可以通过原子替换为nil删除条目：
	// 当下次创建m.dirty时，会原子性地将nil替换为expunged
	// 并保持m.dirty[key]未设置状态
	// 当p!=expunged时，可以通过原子替换更新条目关联的值
	// 如果p==expunged，需要执行m.dirty[key]=e
	// 确保通过dirty map查询能找到该条目后，才能更新关联值
	p atomic.Pointer[any]
}


  

// expunged是一个任意指针，用于标记已从dirty map中删除的条目
var expunged = new(any)

```
**Map结构体**
+ `mu Mutex`.
+ `read atomic.Pointer[readonly]`
	+ read字段包含了map中那些可以安全并发访问的部分内容(无论是否持有mu锁)
	+ read字段本身总是可以安全加载，但更新时必须持有mu锁
	+ read中的条目可以在不持有mu锁的情况下并发更新，但是若要更新一个已经被擦除(expunged)的条目
	+ 需要将该条目复制到dirty map中，并在持有mu锁的情况下取消擦除(unexpunged)
+ `dirty map[any]*entry`
	+ dirty字段保存了map中需要持有mu锁才能访问的部分内容
	+ 为了确保dirty map可以快速提升为read map，还包含了read map中所有未被擦除(unexpunged)的非过期条目
	+ 被擦除(expunged)的条目不会存储在dity map中
	+ 如果要向clean map中的一个被擦除的条目存储新值，必须在持有mu锁的情况下，取消擦除该条目并将其添加到dirty map 中
	+ 如果dirty map为nil，map的下次写入操作会通过浅拷贝clean map(排除过期条目)来初始化它
+ `missed int`
	+ misses计数器记录自read map上次更新后，需要加锁才能判断key是否存在数据访问次数
	+ 当misses达到足够大(足以覆盖复制dirty map的成本时)，dirty map会被提升为read map(保持未修改状态)
	+ 下次map存储操作会创建新的dirty副本


**readonly结构体**
+ `m map[any]*entry`
	+ readOnly是一个不可变(immutable)结构体，以原子方式(atomically)存储在Map.read字段中
+ amended bool
	+ 当dirty map中包含了m中没有的key时返回true

**entry对应的两种删除状态**

| 删除状态     | comment                                                    |     |
| -------- | ---------------------------------------------------------- | --- |
| nil      | 如果p\==nil，表示条目已经被删除，此时m.dirty\==nil 或 m.dirty\[key]等于当前条目e | 软删除 |
| expunged | 如果p\==expunged，表示条目已经被删除，此时m.dirty!=nil，但该条目不存在m.dirty中    | 硬删除 |
+ p指向条目存储的interface{}值
+ 如果p\==nil，表示条目已经被删除，此时m.dirty\==nil 或 m.dirty\[key]等于当前条目e
+ 如果p\==expunged，表示条目已经被删除，此时m.dirty!=nil，但该条目不存在m.dirty中
+ 其它情况下，条目有效且被记录在m.read.m\[key]中，如果m.dirty!=nil,也会记录在m.dirty\[key]中
**可以通过原子替换为nil删除条目**：
+ 当下次创建m.dirty时，会原子性地将nil替换为expunged，并保持m.dirty\[key]未设置状态
+ 当p!=expunged时，可以通过原子替换更新条目关联的值
+ 如果p\==expunged，需要执行m.dirty\[key]=e
+ 确保通过dirty map查询能找到该条目后，才能更新关联值


## 公共辅助函数

**Map的方法**
+ `loadReadOnly`.
	+ 原子加载readonly
	+ 为nil，则新建
+ `missLocked`
	+ 记录在read map中未命中的次数misses
	+ 如果misses等于dirty map的长度，将dirty map提升为read map，dirty map置为nil，重置misses
+ `dirtyLocked`
	+ 将read map中的数据流转到dirty
	+ 只有不是expunged状态的entry才能添加到dirty
**entry的方法**
+ `trySwap`
	+ trySwap在条目未被标记为expunged时执行值交换；若条目已经被标记位expunged，trySwap返回false并保持条目不变
+ `unexpungeLocked`
	+ 清除条目的expunged状态，变为nil状态
	+ 若该条目此前已经被清除，则必须在释放m.mu锁之前将其重新添加到dirty map中
+ `tryExpungeLocked`
	+ 将entry的状态从nil变为expunged
+ `swapLocked`
	+ swapLocked会无条件地将值交换到条目中，必须确保该条目未被标记为已清除(expunged)
+ delete
	+ 如果e.p为nil或者expunged，直接返回nil,false
	+ 第一次删除，将其变为nil


```go
func (m *Map) loadReadOnly() readOnly {
	if p := m.read.Load(); p != nil {
		return *p
	}
	// 懒式创建
	return readOnly{}
}


func (m *Map) missLocked() {
	// 在read map中未命中的次数
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	// 将m.dirty提升为m.read，并将m.dirty置为nil
	m.read.Store(&readOnly{m: m.dirty})
	m.dirty = nil
	// read map被更新，重置misses
	m.misses = 0
}

// trySwap在条目未被标记为expunged时执行值交换
// 若条目已经被标记位expunged，trySwap返回false并保持条目不变
func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		// 加载p
		p := e.p.Load()
		// p的状态为expunged，返回nil,false
		if p == expunged {
			return nil, false
		}
		// 将e.p转换为i，返回p,true
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}



// unexpungeLocked确保该条目未被标记为已清除(exunged)
// 若该条目此前已经被清除，则必须在释放m.mu锁之前将其重新添加到dirty map中
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return e.p.CompareAndSwap(expunged, nil)
}


func (e *entry) tryExpungeLocked() (isExpunged bool) {
	// 加载p
	p := e.p.Load()
	// 如果p是nil，将其转换为expunged
	for p == nil {
		if e.p.CompareAndSwap(nil, expunged) {
			return true
		}
		p = e.p.Load()
	}
	return p == expunged
}
	
// swapLocked会无条件地将值交换到条目中
// 必须确保该条目未被标记为已清除(expunged)
func (e *entry) swapLocked(i *any) *any {
	return e.p.Swap(i)
}


// 将read map中的数据流转到dirty
func (m *Map) dirtyLocked() {
	// 如果m.dirty不为nil，直接返回
	if m.dirty != nil {
		return
	}
	// 加载read
	read := m.loadReadOnly()
	// 新建m.dirty
	m.dirty = make(map[any]*entry, len(read.m))
	// 遍历read.m中的每个key/elem
	for k, e := range read.m {
		// 将nil状态的entry变为expunged状态
		// 只有unexpunged状态的entry才能添加到m.dirty
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}


// 删除一个entry
// 返回值value: 删除之前的值
// ok:删除的键是否存在
func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		// p为nil或者expunged直接返回nil，false
		if p == nil || p == expunged {
			return nil, false
		}
		// 第一次删除将其标记为nil状态
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```
## Load原理

```go
// Load返回存储在map中对应键的值，若不存在则返回nil
// 参数ok表示是否在map中找到了该值
func (m *Map) Load(key any) (value any, ok bool) {
	// 加载m.read
	read := m.loadReadOnly()
	// 访问m.read,不加锁寻找key
	e, ok := read.m[key]
	// 如果没有找到，并且dirty map中包含read中没有的数据
	// 加锁访问dirty
	if !ok && read.amended {
		m.mu.Lock()
		// 再次加载read
		read = m.loadReadOnly()
		// 再次访问read
		e, ok = read.m[key]
		// 如果没有找到，并且dirty map中包含read中没有的数据
		if !ok && read.amended {
			// 访问dirty
			e, ok = m.dirty[key]
			// 增加m.misses，并判断是否需要将dirty提升为read
			m.missLocked()
		}
		// 解锁
		m.mu.Unlock()
	}
	// 没有找到返回nil,false
	if !ok {
		return nil, false
	}
	// 找到了返回对应的值，true
	return e.load()
}
```

**Load流程**
+ 首先不加锁访问read map ，存在直接返回结果
+ 不存在并且dirty 中包含read中没有的数据，加锁
+ 再次访问read，存在返回
+ 不存在且dirty中包含read中没有的数据，访问dirty
	+ 访问dirty，增加m.misses，并判断需不需要将dirty提升为read
+ 最后返回结果
不管加锁还是不加锁，都会优先访问read map
## Store原理

```go
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}



// Swap方法替换键对应的值并返回之前的值（如果存在）。
// loaded返回值用于报告该键是否原本存在。
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	// 加载m.read
	read := m.loadReadOnly()
	// 如果在read中找到key
	if e, ok := read.m[key]; ok {
		// 如果e.p不为expunged状态，修改其值
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				// 该key原先被删除，但是同时存在于read map和dirty map中
				return nil, false
			}
			return *v, true
		}
	}
	// 加锁
	m.mu.Lock()
	// 再次获取m.read
	read = m.loadReadOnly()
	// 如果在read.m中找到key
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// 该条目之前已被清除(expunged)，这意味着存在一个非空的脏映射(dirty map) 
			// 且当前条目不在其中。
			m.dirty[key] = e
		}
		// 更新e的值
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
		// read map中没有找到，去dirty map中找
	} else if e, ok := m.dirty[key]; ok {
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {
		if !read.amended {
			// 正在向脏映射(dirty map)添加首个新键 
			// 需要确保完成分配，并将只读映射标记为不完整
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		// 在dirty map中建立新的映射
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}
```
**Store流程**
+ 首先在read map中寻找key
	+ 若找到，trySwap : 只要e.p不为expunged状态，修改其值
	+ 如果e.p原先为nil，说明被删除但是key同时存在于read 和 dirty ，返回nil, false
+ 加锁，再次在read map中寻找key
	+ 若找到，说明e.p为expunged状态，unexpungeLocked: 将expunged状态变为nil
	+ 重新在dirty map中建立k和e的映射
	+ 更新e的值，并记录返回值
+ 在dirty map中找
	+ 若找到，更新e的值，并记录返回值
+ 如果read.amended为false，说明是第一次写入该键值对
	+ dirtyLocked:将read map中nil状态变为expunged，并将非expunged的数据复制到dirty map(这一步是为了保证dirty map不为nil)
	+ 修改read.amended更改为true
	+ 在dirty map中建立新的映射
+ 解锁，返回返回值

如果在read map中找到的entry是expunged状态，需要先在dirty map中重新建立映射关系，然后修改
如果是首次添加该键，如果read.amended为false，为了保证dirty map不为nil，需要先进行数据流转操作：
将read map中nil的数据变为expunged，并将非expunged的数据添加到dirty map,(如果数据量很大，这一步会非常耗时，而且持有锁的时间会很长)
## Delete原理

```go
// Delete deletes the value for a key.
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key)
}


// LoadAndDelete 删除指定键对应的值，并返回其先前存在的值（如有） 
// 返回值中的 loaded 字段表明该键先前是否存在于映射中
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	// 加载read
	read := m.loadReadOnly()
	// 访问read
	e, ok := read.m[key]
	// 不在read map中，且dirty含有read没有的数据
	if !ok && read.amended {
		// 加锁
		m.mu.Lock()
		// 再次加载read
		read = m.loadReadOnly()
		// 再次访问read map
		e, ok = read.m[key]
		// 不在read map，且dirty包含read没有的数据
		if !ok && read.amended {
			// 访问dirty
			e, ok = m.dirty[key]
			// 这里没有判断是否找到，就执行delete操作
			// 因为delete操作可以重复执行
			delete(m.dirty, key)
			// 无论该条目是否存在（即缓存是否命中），都记录一次未命中（miss） 
			// 该键将在脏映射(dirty map)提升为只读映射(read map)前，一直采用慢速路径(slow path)处理
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		// 如果不是第一次删除(e.p为nil或者expunged)
		// 直接返回nil , false
		// 第一次删除将其变为nil，返回*p,true
		return e.delete()
	}
	return nil, false
}
```
**delete流程**
+ 1.访问read map
	+ 找到key，进入3
+ 2. 没有找到并且dirty map含有read map没有的数据(read.amended为true)，加锁
	+ 再次访问read map，若找到，解锁，进入3
	+ 没有找到且read.amended为true，访问dirty，并将key从dirty中删除
		+ 这里没有判断key是否存在于dirty，是因为delete操作可以重复执行
	+ missLocked:增加m.misses的个数，并判断要不要将dirty提升为read
		+ read未命中就增加m.misses，是为了尽快将dirty提升为read，加快访问速度
	+ 解锁
+ 3. 如果key存在(read or dirty)
	+ delete: 不是第一次删除(e.p为nil，或者expunged)，返回nil，false
	+ 是第一次删除，将其变为nil状态，返回\*p,true
## 总结

一开始写入数据，在dity map中建立新的映射关系，并将read.amended设置为true(表明dirty map中含有read map中没有的数据)；
执行删除操作，在dirty map中将e.p设置为nil状态；(删除操作也会先查找key，所以这里也会修改misses的值，并判断是否需要提升)
然后开始访问，随着misses的个数增加直到等于dirty map的长度，将dirty map提升为read map，重置m.misses和m.read.amended；
此时修改数据，key在read中找到，如果e.p不为expunged直接修改，如果为expunged需要先在dirty map中重新建立映射然后修改；（这一步是为了保证dirty包含所有的映射关系）
此时写入数据，如果amended为false，为了保证dirty 不为nil，需要进行数据流转：将read中nil状态变为expunged，并将non-expunged的数据添加到dirty。之后在dirty中建立新的映射，设置amended为true；

