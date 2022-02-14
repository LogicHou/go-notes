# 10 并发

将程序分成多个可独立执行的部分的结构化程序的设计方法，就是并发设计

采用了并发设计的应用也可以看成是一组独立执行的模块的组合

并发不是并行，并发关乎结构，并行关乎执行

并发是在应用设计与实现阶段要考虑的问题，并发考虑的是如何将应用划分为多个互相配合的、可独立执行的模块的问题

采用并发设计的程序并不一定是并行执行的

# Go 的并发方案：goroutine

Go 并没有使用操作系统线程作为承载分解后的代码片段（模块）的基本执行单元，而是实现了goroutine这一由 Go 运行时（runtime）负责调度的、轻量的用户级线程，为并发程序设计提供原生支持

相比传统操作系统线程来说，goroutine 的优势：

* 资源占用小，每个 goroutine 的初始栈大小仅为 2k
* 由 Go 运行时而不是操作系统调度，goroutine 上下文切换在用户层完成，开销更小
* 在语言层面而不是通过标准库提供，goroutine 由go关键字创建，一退出就会被回收或销毁，开发体验更佳
* 语言内置 channel 作为 goroutine 间通信原语，为并发设计提供了强大支撑

## goroutine 的基本用法

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

### goroutine 的退出

goroutine 的使用代价很低，Go 官方也推荐多多使用 goroutine

而且，多数情况下，我们不需要考虑对 goroutine 的退出进行控制：**goroutine 的执行函数的返回，就意味着 goroutine 退出**

如果 main goroutine 退出了，那么也意味着整个应用程序的退出

goroutine 执行的函数或方法即便有返回值，Go 也会忽略这些返回值

如果要获取 goroutine 执行后的返回值，可以通过 goroutine 间的通信来实现

## goroutine 间的通信

Go 借鉴了 CSP 并发模型，一个符合 CSP 模型的并发程序应该是一组通过输入输出原语连接起来的 P 的集合，P 并不一定与操作系统的进程或线程划等号，在 Go 中，与“Process”对应的是 goroutine

Go 引入了 goroutine（P）之间的通信原语channel

goroutine 可以从 channel 获取输入数据，再将处理后得到的结果数据通过 channel 输出

通过 channel 将 goroutine（P）组合连接在一起，让设计和编写大型并发系统变得更加简单和清晰

用 channel 原语实现获取 goroutine 的退出状态：

    func spawn(f func() error) <-chan error {
        c := make(chan error)

        go func() {
            c <- f()
        }()

        return c
    }

    func main() {
        c := spawn(func() error {
            time.Sleep(2 * time.Second)
            return errors.New("timeout")
        })
        fmt.Println(<-c) // 这里阻塞主了main groutine，等待2秒后接收到错误值后继续向下执行
    }

虽然 CSP 模型已经成为 Go 语言支持的主流并发模型，但 Go 也支持传统的、基于共享内存的并发模型，并提供了基本的低级别同步原语（主要是 sync 包中的互斥锁、条件变量、读写锁、原子操作等）

从程序的整体结构来看，Go 始终推荐以 CSP 并发模型风格构建并发程序，尤其是在复杂的业务层面，能提升程序的逻辑清晰度，大大降低并发设计的复杂性，并让程序更具可读性和可维护性

对于局部情况，比如涉及性能敏感的区域或需要保护的结构体数据时，我们可以使用更为高效的低级同步原语（如 mutex），保证 goroutine 对数据的同步访问