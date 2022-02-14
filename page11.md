# 11 并发原语

## channel 也是一等公民

可以像使用普通变量那样使用 channel，定义 channel 类型变量、给 channel 变量赋值、将 channel 作为参数传递给函数 / 方法、将 channel 作为返回值从函数 / 方法中返回，甚至将 channel 发送到其他 channel 中

### 创建 channel

和切片、结构体、map 等一样，channel 也是一种复合数据类型

在声明一个 channel 类型变量时，必须给出其具体的元素类型：

    var ch chan int

如果 channel 类型变量在声明时没有被赋予初值，那么默认值为 nil

为 channel 类型变量赋初值的唯一方法就是使用 make 这个 Go 预定义的函数：

    ch1 := make(chan int)    // 无缓冲 channel
    ch2 := make(chan int, 5) // 带缓冲 channel
    
使用操作符<-，还可以声明只发送 channel 类型（send-only）和只接收 channel 类型（recv-only）：

    ch1 := make(chan<- int, 1) // 只发送channel类型
    ch2 := make(<-chan int, 1) // 只接收channel类型

    <-ch1       // invalid operation: <-ch1 (receive from send-only type chan<- int)
    ch2 <- 13   // invalid operation: ch2 <- 13 (send to receive-only type <-chan int)


### 发送与接收

通过 <- 操作符用于对 channel 类型变量进行发送与接收操作：

    ch1 <- 13    // 将整型字面值 13 发送到无缓冲 channel 类型变量 ch1 中
    n := <- ch1  // 从无缓冲 channel 类型变量 ch1 中接收一个整型值存储到整型变量 n 中
    ch2 <- 17    // 将整型字面值 17 发送到带缓冲 channel 类型变量 ch2 中
    m := <- ch2  // 从带缓冲 channel 类型变量 ch2 中接收一个整型值存储到整型变量 m 中

根据 <- 符号在通道左右的位置有一个记忆口诀，就是“左接右发”

非常重要的一个概念：**channel 是用于 Goroutine 间通信的**，所以绝大多数对 channel 的读写都被分别放在了不同的 Goroutine 中

#### 通过 for range 接收数据

通过使用 for range 循环语句从 channel 中接收数据，for range 会阻塞在对 channel 的接收操作上，直到 channel 中有数据可接收或 channel 被关闭循环，才会继续向下执行

channel 被关闭后，for range 循环也就结束了

#### 无缓冲 channel

Goroutine 对不带有缓冲区的无缓冲 channel 的接收和发送操作是同步的

也就是说，对同一个无缓冲 channel，只有对它进行接收操作的 Goroutine 和对它进行发送操作的 Goroutine 都存在的情况下，通信才能得以进行，否则单方面的操作会让对应的 Goroutine 陷入挂起状态：

    func main() {
        ch1 := make(chan int)
        ch1 <- 13 // fatal error: all goroutines are asleep - deadlock!
        n := <-ch1
        println(n)
    }

改成这样即可：

    func main() {
        ch1 := make(chan int)
        go func() {
            ch1 <- 13 // 将发送操作放入一个新goroutine中执行
        }()
        n := <-ch1 // 这里阻塞住，等待从上面的 goroutine 中接收数据
        println(n)
    }

**对无缓冲 channel 类型的发送与接收操作，一定要放在两个不同的 Goroutine 中进行，否则会导致 deadlock**

#### 带缓冲 channel

带缓冲 channel 的运行时层实现带有缓冲区，因此，对带缓冲 channel 的发送操作在缓冲区未满、接收操作在缓冲区非空的情况下是**异步**的（发送或接收不需要阻塞等待）

对一个带缓冲 channel 来说：

* 在缓冲区未满的情况下，进行发送操作的 Goroutine 并不会阻塞挂起
* 在缓冲区有数据的情况下，对它进行接收操作的 Goroutine 也不会阻塞挂起
* 当缓冲区满了的情况下，对它进行发送操作的 Goroutine 就会阻塞挂起
* 当缓冲区为空的情况下，对它进行接收操作的 Goroutine 也会阻塞挂起

示例：

    ch2 := make(chan int, 1)
    n := <-ch2 // 由于此时ch2的缓冲区中无数据，因此对其进行接收操作将导致goroutine挂起

    ch3 := make(chan int, 1)
    ch3 <- 17  // 向ch3发送一个整型数17
    ch3 <- 27  // 由于此时ch3中缓冲区已满，再向ch3发送数据也将导致goroutine挂起

### 关闭 channel

channel 关闭后，所有等待从这个 channel 接收数据的操作都将返回

采用不同接收语法形式的语句，在 channel 被关闭后的返回值的情况：

    n := <- ch      // 当ch被关闭后，n将被赋值为ch元素类型的零值
    m, ok := <-ch   // 当ch被关闭后，m将被赋值为ch元素类型的零值, ok值为false
    for v := range ch { // 当ch被关闭后，for range循环结束
        ... ...
    }

通过“comma, ok”惯用法或 for range 语句，可以准确地判定 channel 是否被关闭

而单纯采用n := <-ch形式的语句，就无法判定从 ch 返回的元素类型零值，究竟是不是因为 channel 被关闭后才返回的

**在发送端负责关闭 channel ，是 channel 的一个使用惯例**

这是因为发送端没有像接受端那样的、可以安全判断 channel 是否被关闭了的方法

同时，一旦向一个已经关闭的 channel 执行发送操作，这个操作就会引发 panic ：

    ch := make(chan int, 5)
    close(ch)
    ch <- 13 // panic: send on closed channel

### select

通过 select，可以同时在多个 channel 上进行发送 / 接收操作：

    select {
    case x := <-ch1:     // 从channel ch1接收数据
      ... ...

    case y, ok := <-ch2: // 从channel ch2接收数据，并根据ok值判断ch2是否已经关闭
      ... ...

    case ch3 <- z:       // 将z值发送到channel ch3中:
      ... ...

    default:             // 当上面case中的channel通信均无法实施时，执行该默认分支
    }

当 select 语句中没有 default 分支，而且所有 case 中的 channel 操作都阻塞了的时候，整个 select 语句都将被阻塞，直到某一个 case 上的 channel 变成可发送，或者某个 case 上的 channel 变成可接收，select 语句才可以继续进行下去

### 无缓冲 channel 的惯用法

#### 第一种用法：用作信号传递

无缓冲 channel 用作信号传递的时候，有两种情况，分别是 1 对 1 通知信号和 1 对 n 通知信号

1 对 1 的情况下，可以通过在一个通道内返回一个专门用作通知 main goroutine 的信号来结束阻塞动作

1 对 n 的情况下，这样的信号通知机制，常被用于协调多个 Goroutine 一起工作，通过在多个 goroutine 中植入信号接收操作阻塞住各个 goroutine 的执行，然后通过在 main goroutine 中 close 通道解除阻塞是 goroutine 中的逻辑继续执行

#### 第二种用法：用于替代锁机制

无缓冲 channel 具有同步特性，这让它在某些场合可以替代锁，让我们的程序更加清晰，可读性也更好

比如可以将并发的计数器操作全部交给一个独立的 Goroutine 去处理，并通过无缓冲 channel 的同步阻塞特性，实现了计数器的控制

这样其他 Goroutine 通过增加计数器值的动作，实质上就转化为了一次无缓冲 channel 的接收动作

这种并发设计逻辑更符合 Go 语言所倡导的“**不要通过共享内存来通信，而是通过通信来共享内存**”的原则

### 带缓冲 channel 的惯用法

带缓冲的 channel 与无缓冲的 channel 的最大不同之处，就在于它的异步性

对于一个带缓冲 channel，在缓冲区未满的情况下，对它进行发送操作的 Goroutine 不会阻塞挂起

在缓冲区有数据的情况下，对它进行接收操作的 Goroutine 也不会阻塞挂起

#### 第一种用法：用作消息队列

和无缓冲 channel 更多用于信号 / 事件管道相比，可自行设置容量、异步收发的带缓冲 channel 更适合被用作为消息队列，并且，带缓冲 channel 在数据收发的性能上要明显好于无缓冲 channel

不过你 Go 支持 channel 的初衷是作为 Goroutine 间的通信手段，并不是专门用于消息队列场景的

如果项目需要专业消息队列的功能特性，比如支持优先级、支持权重、支持离线持久化等，那么 channel 就不合适了，可以使用第三方的专业的消息队列实现

#### 第二种用法：用作计数信号量（counting semaphore）

带缓冲 channel 中的当前数据个数代表的是(一种假设)，当前同时处于活动状态（处理业务）的 Goroutine 的数量，而带缓冲 channel 的容量（capacity），就代表了允许同时处于活动状态的 Goroutine 的最大数量

向带缓冲 channel 的一个发送操作表示获取一个信号量，而从 channel 的一个接收操作则表示释放一个信号量

