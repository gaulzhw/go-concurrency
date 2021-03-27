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



### map for thread safe



### Pool



### Context



## atomic



## channel

### channel




### 内存模型



## 扩展并发原语



## 分布式并发原语



## Ref

[64位对齐](https://go101.org/article/memory-layout.html)
