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

### RWMutex

### WaitGroup

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
