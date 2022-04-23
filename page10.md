# 10 并发

将程序分成多个可独立执行的部分的结构化程序的设计方法，就是并发设计

采用了并发设计的应用也可以看成是一组独立执行的模块的组合

并发不是并行，并发关乎结构，并行关乎执行

并发是在应用设计与实现阶段要考虑的问题，并发考虑的是如何将应用划分为多个互相配合的、可独立执行的模块的问题

采用并发设计的程序并不一定是并行执行的

在仅有一个单核 CPU 的情况下，即便是采用并发设计的程序，依旧不可以并行执行。而在满足并行必要条件的情况下，采用并发设计的程序是可以并行执行的。而那些没有采用并发设计的应用程序，除非是启动多个程序实例，否则是无法并行执行的。

# Go 的并发方案：goroutine

Go 并没有使用操作系统线程作为承载分解后的代码片段（模块）的基本执行单元，而是实现了goroutine这一**由 Go 运行时（runtime）负责调度的、轻量的用户级线程**，为并发程序设计提供原生支持

相比传统操作系统线程来说，goroutine 的优势：

* 资源占用小，每个 goroutine 的初始栈大小仅为 2k
* 由 Go 运行时而不是操作系统调度，goroutine 上下文切换在用户层完成，开销更小
* 在语言层面而不是通过标准库提供，goroutine 由go关键字创建，一退出就会被回收或销毁，开发体验更佳
* 语言内置 channel 作为 goroutine 间通信原语，为并发设计提供了强大支撑

Go 语言是面向并发而生的，所以，在程序的结构设计阶段，**Go 的惯例是优先考虑并发设计**，可以更好地、更自然地适应规模化（scale）

## goroutine 的基本用法

并发是一种能力，让程序可以由若干个代码片段组合而成，并且每个片段都是独立运行的

goroutine 是 Go 原生支持并发的一个具体实现，无论是 Go 自身运行时代码还是用户层 Go 代码，都无一例外地运行在 goroutine 中

Go 语言通过go关键字+函数/方法的方式创建一个 goroutine

创建后，新 goroutine 将拥有独立的代码执行流，并与创建它的 goroutine 一起被 Go 运行时调度：

    go fmt.Println("I am a goroutine")

    var c = make(chan int)
    go func(a, b int) {
        c <- a + b
    }(3,4)
    
    // $GOROOT/src/net/http/server.go
    c := srv.newConn(rw)
    go c.serve(connCtx)

通过 go 关键字，可以基于已有的具名函数 / 方法创建 goroutine，也可以基于匿名函数 / 闭包创建 goroutine

和线程一样，一个应用内部启动的所有 goroutine 共享进程空间的资源，如果多个 goroutine 访问同一块内存数据，将会存在竞争，需要进行 goroutine 间的同步

### goroutine 的退出

goroutine 的使用代价很低，Go 官方也推荐多多使用 goroutine

多数情况下，不需要考虑对 goroutine 的退出进行控制：**goroutine 的执行函数的返回，就意味着 goroutine 退出**

如果 main goroutine 退出了，那么也意味着整个应用程序的退出

goroutine 执行的函数或方法即便有返回值，Go 也会忽略这些返回值

如果要获取 goroutine 执行后的返回值，可以通过 goroutine 间的通信来实现

## goroutine 间的通信

Go 引入了 goroutine 之间的通信原语 channel

goroutine 可以从 channel 获取输入数据，再将处理后得到的结果数据通过 channel 输出

通过 channel 将 goroutine 组合连接在一起，让设计和编写大型并发系统变得更加简单和清晰

用 channel 原语实现获取 goroutine 的退出状态的例子：

    func spawn(f func() error) <-chan error {
        c := make(chan error)

        go func() {    // 新创建了一个goroutine，注意这里是不会阻塞return c的执行的
            c <- f()   // 2秒后将回调函数的值塞进通道里
        }()

        return c       // 返回了chan
    }

    func main() {
        c := spawn(func() error {
            time.Sleep(2 * time.Second)
            return errors.New("timeout")
        })
        fmt.Println(<-c) // 这里阻塞主了main groutine，等待2秒后接收到错误值后继续向下执行
    }

Go 也支持传统的、基于共享内存的并发模型，并提供了基本的低级别同步原语（主要是 sync 包中的互斥锁、条件变量、读写锁、原子操作等）

**Go 始终推荐以 CSP 并发模型风格构建并发程序**，尤其是在复杂的业务层面，能提升程序的逻辑清晰度，大大降低并发设计的复杂性，并让程序更具可读性和可维护性

不过对于局部情况，比如涉及性能敏感的区域或需要保护的结构体数据时，我们可以使用更为高效的低级同步原语（如 mutex），保证 goroutine 对数据的同步访问

## goroutine 调度器

Go 语言中的并发实现，使用了 Goroutine，代替了操作系统的线程，也不再依靠操作系统调度

Goroutine 占用的资源非常小，每个 Goroutine 栈的大小默认是 2KB

Goroutine 调度的切换也不用陷入（trap）操作系统内核层完成，代价很低

将这些 Goroutine 按照一定算法放到“CPU”上执行的程序，就被称为 Goroutine 调度器（Goroutine Scheduler）

Go 程序内 Goroutine 之间“公平”竞争“CPU”资源的任务，由 Go 运行时（runtime）负责

在操作系统层面，线程竞争的“CPU”资源是真实的物理 CPU，但在 Go 程序层面，各个 Goroutine 要竞争的“CPU”资源就是操作系统线程

Goroutine 调度器任务是：**将 Goroutine 按照一定算法放到不同的操作系统线程中去执行**

### Goroutine 调度器模型与演化过程

G、P 、M抽象结构的说明：

* G(Goroutine)  ：代表 Goroutine，存储了 Goroutine 的执行栈信息、Goroutine 状态以及 Goroutine 的任务函数等，而且 G 对象是可以重用的
* P(Proccessor) ：代表逻辑 processor，P 的数量决定了系统内最大可并行的 G 的数量，P 的最大作用还是其拥有的各种 G 对象队列、链表、一些缓存和状态
* M(machine)    ：代表着真正的执行计算资源。在绑定有效的 P 后，进入一个调度循环，而调度循环的机制大致是从 P 的本地运行队列以及全局队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M，如此反复。M 并不保留 G 状态，这是 G 可以跨 M 调度的基础

从最初的 G-M 模型、到 G-P-M 模型，从不支持抢占，到支持协作式抢占，再到支持基于信号的异步抢占

### G-M 调度模型（已经废弃）

此时的调度器的工作就是将 G 调度到 M 上去运行

为了更好地控制程序中活跃的 M 的数量，调度器引入了 GOMAXPROCS 变量来表示 Go 调度器可见的“处理器”的最大数量

遇到的问题：

* 单一全局互斥锁(Sched.Lock) 和集中状态存储的存在，导致所有 Goroutine 相关操作，比如创建、重新调度等，都要上锁
* Goroutine 传递问题：M 经常在 M 之间传递“可运行”的 Goroutine，这导致调度延迟增大，也增加了额外的性能损耗
* 每个 M 都做内存缓存，导致内存占用过高，数据局部性较差
* 由于系统调用（syscall）而形成的频繁的工作线程阻塞和解除阻塞，导致额外的性能损耗

### G-P-M 调度模型

每个 G（Goroutine）首先需要被分配一个 P，进入到 P 的本地运行队列（local runq）中

对于 G 来说，P 就是运行它的“CPU”，可以说：在 G 的眼里只有 P

但从 Go 调度器的视角来看，真正的“CPU”是 M，只有将 P 和 M 绑定，才能让 P 的 runq 中的 G 真正运行起来

初期的 G-P-M 模型实现**不支持抢占式调度**，这导致一旦某个 G 中出现死循环的代码逻辑，那么 G 将永久占用分配给它的 P 和 M，而位于同一个 P 中的其他 G 将得不到调度，特别是当只有一个 P（GOMAXPROCS=1）时，整个 Go 程序中的其他 G 都将“卡住”

**Go 1.2 中实现了基于协作的“抢占式”调度**

这个抢占式调度的原理就是，Go 编译器在每个函数或方法的入口处加上了一段额外的代码 (runtime.morestack_noctxt)，让运行时有机会在这段代码中检查是否需要执行抢占调度

这种方式只能说局部解决了“卡住”问题，只在有函数调用的地方才能插入“抢占”代码（埋点），对于没有函数调用而是纯算法循环计算的 G，Go 调度器依然无法抢占

**Go 在 1.14 版本中增加了对非协作的抢占式调度的支持**

这种抢占式调度是基于系统信号的，也就是通过向线程发送信号的方式来抢占正在运行的 Goroutine

