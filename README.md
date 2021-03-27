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
- 常见的4种错误场景
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
- 常见的3种错误场景
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



### Cond



### Once



### map for thread safe



### Pool



### Context



## atomic



## channel

### channel


### 内存模型



## 扩展并发原语



## 分布式并发原语
