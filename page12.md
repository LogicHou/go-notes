# 轻量级线程池的实现

## Goroutine 的性能问题

**Goroutine 的开销虽然“廉价”，但也不是免费的**，一旦规模化后，这种非零成本也会成为瓶颈

Goroutine 从Go 1.4 版本开始采用了连续栈的方案，也就是每个 Goroutine 的执行栈都是一块连续内存，如果空间不足，运行时会分配一个更大的连续内存空间作为这个 Goroutine 的执行栈，将原栈内容拷贝到新分配的空间中来

连续栈的方案，虽然能避免 Go 1.3 采用的分段栈会导致的“hot split”问题，但连续栈的原理也决定了，一旦 Goroutine 的执行栈发生了 grow，那么即便这个 Goroutine 不再需要那么大的栈空间，这个 Goroutine 的栈空间也不会被 Shrink（收缩）了，这些空间可能会处于长时间闲置的状态，直到 Goroutine 退出

随着 Goroutine 数量的增加，Go 运行时进行 Goroutine 调度的处理器消耗，也会随之增加，成为阻碍 Go 应用性能提升的重要因素

**常见的解决方案是 Goroutine 池**

这个方案的核心思想是对 Goroutine 的重用，也就是把 M 个计算任务调度到 N 个 Goroutine 上，而不是为每个计算任务分配一个独享的 Goroutine，从而提高计算资源的利用率

### 线程池 workerpool 的实现思路

workerpool 的实现主要分为三个部分：

* pool 的创建与销毁
* pool 中 worker（Goroutine）的管理
* task 的提交与调度

原理：

pool 有一个 capacity 的属性，代表整个 pool 中 worker 的最大容量。使用一个带缓冲的 channel：active，作为 worker 的“计数器”，也就 channel **计数信号量**的使用模式

当 active channel 可写时，就创建一个 worker，用于处理用户通过 Schedule 函数提交的待处理的请求。当 active channel 满了的时候，pool 就会停止 worker 的创建，直到某个 worker 因故退出，active channel 又空出一个位置时，pool 才会创建新的 worker 填补那个空位

把用户要提交给 workerpool 执行的请求抽象为一个 Task。Task 的提交与调度也很简单：Task 通过 Schedule 函数提交到一个 task channel 中，已经创建的 worker 将从这个 task channel 中读取 task 并执行

一段 workpool 源码解析：

    package workerpool

    import (
      "errors"
      "fmt"
      "sync"
    )

    var (
      ErrNoIdleWorkerInPool = errors.New("no idle worker in pool") // workerpool中任务已满，没有空闲goroutine用于处理新任务
      ErrWorkerPoolFreed    = errors.New("workerpool freed")       // workerpool已终止运行
    )

    type Pool struct {
      capacity int         // workerpool大小

      active chan struct{} // 对应active channel
      tasks  chan Task     // 对应task channel

      wg   sync.WaitGroup  // 用于在pool销毁时等待所有worker退出
      quit chan struct{}   // 用于通知各个worker退出的信号channel
    }

    type Task func()       // 定义Task类型为函数类型

    const (
      defaultCapacity = 100
      maxCapacity     = 10000
    )

    // 用于创建一个 pool 类型实例，并将 pool 池的 worker 管理机制运行起来
    func New(capacity int) *Pool {
      // 对 capacity 参数的“防御性”校验 start
      if capacity <= 0 {
        capacity = defaultCapacity
      }
      if capacity > maxCapacity {
        capacity = maxCapacity
      }
      // 对 capacity 参数的“防御性”校验 end

      
      p := &Pool{                                 // 定义pool类型的实例
        capacity: capacity,                       // pool容量
        tasks:    make(chan Task),                // 存tasks的channel，无缓冲区channel
        quit:     make(chan struct{}),            // 用于传递退出信号的channel，无缓冲区channel
        active:   make(chan struct{}, capacity),  // 存放worker的active channel，是个带capacity大小的缓冲区channel
      }

      fmt.Printf("workerpool start\n")

      go p.run()

      return p
    }

    func (p *Pool) newWorker(i int) {
      p.wg.Add(1)
      go func() {
        // worker因故退出部分的代码 start
        defer func() { 
          if err := recover(); err != nil {
            fmt.Printf("worker[%03d]: recover panic[%s] and exit\n", i, err)
            <-p.active // 更新了 worker 计数器
          }
          p.wg.Done()  // 减少 WaitGroup 的 Goroutine 等待数量
        }()
        // worker因故退出部分的代码 end

        fmt.Printf("worker[%03d]: start\n", i)

        for {
          select {
          case <-p.quit:
            fmt.Printf("worker[%03d]: exit\n", i)
            <-p.active
            return
          case t := <-p.tasks:  // 这里worker从tasks队列中拿任务
            fmt.Printf("worker[%03d]: receive a task\n", i)
            t()                 // 运行任务，任务运行完后继续循环拿下一个任务执行
          }
        }
      }()
    }

    func (p *Pool) run() {
      idx := 0

      for {
        select {
        case <-p.quit:
          return
        case p.active <- struct{}{}:  // 这里会不断循环填满active channel，并创建相应数量的worker
          // create a new worker
          idx++
          p.newWorker(idx)            // 创建worker，这里创建的worker除非panic了否则会一直存在
        }
      }
    }

    // 这是 Pool 类型的一个导出方法，workerpool 包的用户通过该方法向 pool 池提交待执行的任务（Task）
    func (p *Pool) Schedule(t Task) error {
      select {
      case <-p.quit:
        return ErrWorkerPoolFreed
      case p.tasks <- t:  // 往tasks中加任务的地方，这里的 Pool 结构体中的 tasks 是一个无缓冲的 channel，如果 pool 中 worker 数量已达上限，而且 worker 都在处理 task 的状态，那么 Schedule 方法就会阻塞，直到有 worker 变为 idle 状态来读取 tasks channel，schedule 的调用阻塞才会解除
        return nil
      }
    }

    // 用于销毁一个 pool 池，停掉所有 pool 池中的 worker
    func (p *Pool) Free() {
      close(p.quit) // 确保所有的worker和p.run退出，并且也结束schedule并返回错误，这里的实现原理就是关闭p.quit通道后，上面代码中case <-p.quit部分的代码都会从关闭的通道中接受数据不会再被阻塞会读取到其零值，这么以来就可以执行case <-p.quit:后面的逻辑了
      p.wg.Wait()
      fmt.Printf("workerpool freed\n")
    }

调用示例：

    package main
      
    import (
        "time"
        "github.com/bigwhite/workerpool"
    )

    func main() {
        p := workerpool.New(5) // 创建了一个 capacity 为 5 的 workerpool 实例

        for i := 0; i < 10; i++ { // 连续向这个 workerpool 提交了 10 个 task
            err := p.Schedule(func() {
                time.Sleep(time.Second * 3) // 每个 task 的逻辑很简单，只是 Sleep 3 秒后就退出
            })
            if err != nil {
                println("task: ", i, "err:", err)
            }
        }

        // 提交完任务后，调用 workerpool 的 Free 方法销毁 pool，pool 会等待所有 worker 执行完 task 后再退出(因为Schedule的阻塞所以产生了等待)
        p.Free()
    }