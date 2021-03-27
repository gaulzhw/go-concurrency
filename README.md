# go-concurrency

go并发主要有两个方向：channel、并发原语
- 共享资源：并发地读写共享资源，会出现数据竞争（data race）的问题，所以需要 Mutex、RWMutex 这样的并发原语来保护
- 任务编排：需要 goroutine 按照一定的规律执行，而 goroutine 之间有相互等待或者依赖的顺序关系，我们常常使用 WaitGroup 或者 Channel 来实现
- 消息传递：信息交流以及不同的 goroutine 之间的线程安全的数据交流，常常使用 Channel 来实现



## 并发原语

### mutex
- 使用 demo.go
- 竞态分析
  - go run -race xx.go
  - go tool compile -race -S xx.go
- vet静态代码检查，go vet xx.go，可以检查出死锁
- 第三方的死锁检测工具
  - go-deadlock (https://github.com/sasha-s/go-deadlock)
  - go-tools (https://github.com/dominikh/go-tools)
- 常见错误场景
  - Lock/Unlock不是成对出现
  - Copy已使用的mutex
  - mutex不是可重入锁
  - 死锁
- 扩展功能
  - TryLock
  - 等待者的数量
  - 线程安全的队列

![state](images/mutex_state.jpg)

```go
type Mutex struct {
  state int32
  sema uint32
}

const (
  mutexLocked = 1 << iota // mutex is locked
  mutexWoken
  mutexStarving // 从state字段中分出一个饥饿标记
  mutexWaiterShift = iota
  starvationThresholdNs = 1e6
)
```



### RWMutex

基于Mutex实现
- 使用
  - Lock/Unlock
  - RLock/RUnlock
  - RLocker
- 适合场景：读多写少
- 基于对读和写操作的优先级，读写锁的设计与实现，Go的RWMutex设计是Write-preferring方案，一个正在阻塞的lock调用会排除新的reader请求到锁
  - Read-preferring：读优先的设计可以提供很高的并发性，但是在竞争激烈的情况下可能会导致写饥饿
  - Write-preferring：写优先的设计意味着如果已经有一个writer在等待请求锁的话，它会阻止新来的请求锁的reader获取到锁，所以优先保障writer
  - 不指定优先级：不区分reader和writer优先级
- 常见错误场景
  - 不可复制
  - 重入导致死锁（产生原因是Go的RWMutex是Write-preferring）
    - 读写锁因为重入（或递归调用）导致死锁
    - 在reader的读操作时调用writer的写操作，这个reader和writer会形成互相依赖的死锁状态
    - 一个writer请求锁的时候如果已经有一些活跃的reader，它会等待这些活跃的reader完成才能获取到锁，但如果之后活跃的reader再以来新的reader的话这些新的reader就会等待writer释放锁之后才能继续执行，就形成一个环形依赖：writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer
  - 释放未加锁的RWMutex

```go
type RWMutex struct {
  w Mutex // 互斥锁解决多个writer的竞争
  writerSem uint32 // writer信号量
  readerSem uint32 // reader信号量
  readerCount int32 // reader的数量
  readerWait int32 // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30
```



### WaitGroup

解决 并发-等待 问题

- 正确姿势
  - 预先确定好WaitGroup的计数值，然后调用相同次数的Done完成相应的任务

- 常见错误场景
  - 计数器设置为负值，WaitGroup的计数器的值必须大于等于0
    - 调用Add的时候传递一个负数
    - 调用Done方法的次数过多，超过了WaitGroup的计数值
  - 不期望的Add时机，原则：等所有的Add方法调用之后再调用Wait，否则可能导致panic或者不期望的结果
  - 前一个Wait还没结束就重用WaitGroup

```go
type WaitGroup struct {
  // 避免复制使用的一个技巧，可以告诉vet工具违反了复制使用的规则
  noCopy noCopy
  // 64bit(8bytes)的值分成两段，高32bit是计数值，低32bit是waiter的计数
  // 另外32bit是用作信号量的
  // 因为64bit值的原子操作需要64bit对齐，但是32bit编译器不支持，所以数组中的元素在不同的架构中不一样，具体处理看下面的方法
  // 总之，会找到对齐的那64bit作为state，其余的32bit做信号量
  state1 [3]uint32
}
```



### Cond

为 等待/通知 场景下的并发问题提供支持

在实践中，处理等待/通知的场景时，常常使用Channel替换Cond，因为Channel类型使用起来更简洁，而且不容易出错。但是对于需要重复调用Broadcast的场景，使用Cond就再合适不过了。

调用Wait方法前必须要持有锁

Signal、Broadcast不强求要持有锁

- 常见错误场景
  - 调用Wait的时候没有加锁
  - 没有检查条件是否满足程序就继续执行

Cond有三点特性是Channel无法替代的：

- Cond和一个Locker关联，可以利用这个Locker对相关的依赖条件更改提供保护
- Cond可以同时支持Signal和Broadcast方法，而Channel只能同时支持其中一个（Channel实现Broadcast可以通过close来实现）
- Cond的Broadcast方法可以被重复调用。等待条件再次变成不满足的状态后，我们又可以调用Broadcast再次唤醒等待的goroutine。这也是Channel不能支持的，Channel被close掉之后不支持再open



### Once

Once可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景

- 常见错误场景
  - 死锁，Once的Do方法的f参数中不要调用当前这个Once
  - 未初始化，f方法执行的时候panic或者f执行初始化资源失败了，Once还会认为初次执行已经成功



### map & sync.Map

并发map更高效的使用方式：分片+加锁（尽量减少锁的粒度和锁的持有时间）

- map的常见错误
  - 未初始化
  - 并发读写

https://github.com/elliotchance/orderedmap

https://github.com/orcaman/concurrent-map

```go
var SHARD_COUNT = 32 

// 分成SHARD_COUNT个分片的
map type ConcurrentMap []*ConcurrentMapShared

// 通过RWMutex保护的线程安全的分片，包含一个map
type ConcurrentMapShared struct {
  items map[string]interface{}
  sync.RWMutex
  // Read Write mutex, guards access to internal map.
}

// 创建并发map
func New() ConcurrentMap {
  m := make(ConcurrentMap, SHARD_COUNT)
  for i := 0; i < SHARD_COUNT; i++ {
    m[i] = &ConcurrentMapShared{items: make(map[string]interface{})}
  }
  return m
}

// 根据key计算分片索引
func (m ConcurrentMap) GetShard(key string) *ConcurrentMapShared {
  return m[uint(fnv32(key))%uint(SHARD_COUNT)]
}
```

- sync.Map使用场景
  - 只会增长的缓存系统中，一个key只写入一次而被读很多次
  - 多个goroutine为不相交的键集读、写和重写键值对

- sync.Map实现有几个优化点
  - 空间换时间，通过冗余的两个数据结构（只读的read、可写的dirty）来减少加锁对性能的影响，对只读字段的操作不需要加锁
  - 优先从read字段读取、更新、删除，因为对read字段的读取不需要锁
  - 动态调整，miss次数多了之后，将dirty数据提升为read，避免总是从dirty中加锁读取
  - double-checking，加锁之后先还要再检查read字段，确定真的不存在才操作diryt字段
  - 延迟删除，删除一个键值只是打标记，只有在提升dirty字段为read字段的时候才清理删除的数据

```go
type Map struct {
  mu Mutex
  // 基本上你可以把它看成一个安全的只读的map
  // 它包含的元素其实也是通过原子操作更新的，但是已删除的entry就需要加锁操作了
  read atomic.Value // readOnly
  
  // 包含需要加锁才能访问的元素
  // 包括所有在read字段中但未被expunged（删除）的元素以及新加的元素
  dirty map[interface{}]*entry
  
  // 记录从read中读取miss的次数，一旦miss数和dirty长度一样了，就会把dirty提升为read，并把dirty置空
  misses int
}

type readOnly struct {
  m map[interface{}]*entry
  amended bool // 当dirty中包含read没有的数据时为true，比如新增一条数据
}

// expunged是用来标识此项已经删掉的指针
// 当map中的一个项目被删除了，只是把它的值标记为expunged，以后才有机会真正删除此项
var expunged = unsafe.Pointer(new(interface{}))

// entry代表一个值
type entry struct {
  p unsafe.Pointer // *interface{}
}
```

https://github.com/zekroTJA/timedmap

https://godoc.org/github.com/emirpasic/gods/maps/treemap



### Pool

用来保存一组可独立访问的临时对象，它池化的对象会在未来的某个时候被毫无征兆地移除掉，如果没有别的对象引用这个被移除的对象的话，这个被移除的对象会被垃圾回收掉

- sync.Pool本身就是线程安全的，多个goroutine可以并发地调用它的方法存取对象
- sync.Pool不可在使用之后再复制使用

- sync.Pool的坑
  - 内存泄漏
  - 内存浪费

第三方库

- bytebufferpool https://github.com/valyala/bytebufferpool
- bpool https://github.com/oxtoacart/bpool

更多池化场景

- 连接池
- http client池
- tcp连接池 https://github.com/fatih/pool
- 数据库连接池
- memcached client连接池 https://github.com/bradfitz/gomemcache
- worker pool
  - 大部分的worker pool都是通过Channel来缓存任务的，因为Channel能够比较方便地实现并发的保护
  - 有的是多个Worker共享同一个任务Channel
  - 有些每个Worker都有一个独立的Channel
  - https://github.com/valyala/fasthttp/blob/9f11af296864153ee45341d3f2fe0f5178fd6210/workerpool.go#L16
  - https://godoc.org/github.com/gammazero/workerpool 可以无限制地提交任务，提供了更便利的 Submit 和 SubmitWait 方法提交任务，还可以提供当前的 worker 数和任务数以及关闭 Pool 的功能
  - https://godoc.org/github.com/ivpusic/grpool 创建 Pool 的时候需要提供 Worker 的数量和等待执行的任务的最大数量，任务的提交是直接往 Channel 放入任务
  - https://pkg.go.dev/github.com/dpaks/goworkers 提供了更便利的 Submit 方法提交任务以及 Worker 数、任务数等查询方法、关闭 Pool 的方法。它的任务的执行结果需要在 ResultChan 和 ErrChan 中去获取，没有提供阻塞的方法，但是它可以在初始化的时候设置 Worker 的数量和任务数
  - https://github.com/panjf2000/ants
  - https://github.com/Jeffail/tunny
  - https://github.com/benmanns/goworker
  - https://github.com/go-playground/pool
  - https://github.com/Sherifabdlnaby/gpool
  - https://github.com/alitto/pond



### Context

- 常用场景
  - 上下文信息传递，比如处理http请求、在请求处理链路上传递信息
  - 控制子goroutine的运行
  - 超时控制的方法调用
  - 可以取消的方法调用



## atomic

适用场景：不涉及到对资源复杂的竞争逻辑，只是会并发地读写这个标志

https://github.com/uber-go/atomic

atomic包提供的方法会提供内存屏障的功能，所以atomic不仅仅可以保证赋值的数据完整性，还能保证数据的可见性，一旦一个核更新了该地址的值，其他处理器总是能读取到它的最新值。但是，因为需要处理器之间保证数据的一致性，atomic的操作也是会降低性能的。

https://github.com/golang/go/issues/39351



## channel

### channel




### 内存模型



## 扩展并发原语



## 分布式并发原语



## Ref

[64位对齐](https://go101.org/article/memory-layout.html)

[Atomic vs. Non-Atomic Operations](https://preshing.com/20130618/atomic-vs-non-atomic-operations/)

[Lockless Programming Considerations for Xbox 360 and Microsoft Windows](https://docs.microsoft.com/zh-cn/windows/win32/dxtecharts/lockless-programming)