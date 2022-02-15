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

从一个已关闭的 channel 接收数据将永远不会被阻塞

对一个 nil channel 执行获取操作，这个操作将阻塞

#### 通过 for range 接收数据

通过使用 for range 循环语句从 channel 中接收数据，for range 会阻塞在对 channel 的接收操作上，直到 channel 中有数据可接收或 channel 被关闭循环，才会继续向下执行

channel 被关闭后，for range 循环也就结束了，如果 channel 里有数据，for range 会接收完里面的数据再结束：

    var jobs = make(chan int, 10)

    func main() {
      go func() {
        for i := 0; i < 8; i++ {
          jobs <- (i + 1)
        }
        close(jobs)
      }()

      for j := range jobs {
        fmt.Println(j) // 输出 1 2 3 4 5 6 7 8
      }
    }

### 无缓冲 channel

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

### 带缓冲 channel

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

### len(channel) 的应用

len 是 Go 语言的一个内置函数，支持接收数组、切片、map、字符串和 channel 类型的参数，并返回对应类型的“长度”，也就是一个整型值

针对 channel ch 的类型不同，len(ch) 有如下两种语义：

* 当 ch 为无缓冲 channel 时，len(ch) 总是返回 0，这里调用 len(ch) 就没什么意义
* 当 ch 为带缓冲 channel 时，len(ch) 返回当前 channel ch 中**尚未被读取**的元素个数

可以使用 len 函数来实现带缓冲 channel 的“判满”、“判有”和“判空”逻辑：

    var ch chan T = make(chan T, capacity)

    // 判空
    if len(ch) == 0 {
        // 此时channel ch空了?
    }

    // 判有
    if len(ch) > 0 {
        // 此时channel ch中有数据?
    }

    // 判满
    if len(ch) == cap(ch) {
        // 此时channel ch满了?
    }

channel 原语用于多个 Goroutine 间的通信，一旦多个 Goroutine 共同对 channel 进行收发操作，len(channel) 就会在多个 Goroutine 间形成“竞态”

单纯地依靠 len(channel) 来判断 channel 中元素状态，是不能保证在后续对 channel 的收发时 channel 状态是不变的

因此，常见的方法是将“判空与读取”放在一个“事务”中，将“判满与写入”放在一个“事务”中，而这类“事务”可以通过 select 实现

这种方法适用于大多数场合，但是这种方法有一个“问题”，那就是它改变了 channel 的状态，会让 channel 接收了一个元素或发送一个元素到 channel

如果想要单纯地侦测 channel 的状态，而又不会因 channel 满或空阻塞在 channel 上，目前没有一种方法可以在实现这样的功能的同时，适用于所有场合

但是在特定的场景下，可以用 len(channel) 来实现，比如下面这两种场景：

* 是一个“多发送单接收”的场景，也就是有多个发送者，但有且只有一个接收者。在这样的场景下，可以在接收 goroutine 中使用len(channel)是否大于0来判断是否 channel 中有数据需要接收 if len(c) > 0
* 是一个“多接收单发送”的场景，也就是有多个接收者，但有且只有一个发送者。在这样的场景下，可以在发送 Goroutine 中使用len(channel)是否小于cap(channel)来判断是否可以执行向 channel 的发送操作 if len(c) < cap(c)

### nil channel

如果一个 channel 类型变量的值为 nil，称为 nil channel，赋初值需要使用 make

nil channel 有一个特性，那就是对 nil channel 的读写都会发生阻塞：

    func main() {
      var c chan int
      <-c //阻塞
    }

    或者：

    func main() {
      var c chan int
      c<-1  //阻塞
    }

### 与 select 结合使用的一些惯用法

#### 利用 default 分支避免阻塞

select 语句的 default 分支的语义，就是在其他非 default 分支因通信未就绪，而无法被选择的时候执行的，这就给 default 分支赋予了一种“避免阻塞”的特性

    // $GOROOT/src/time/sleep.go
    func sendTime(c interface{}, seq uintptr) {
        // 无阻塞的向c发送当前时间
        select {
        case c.(chan Time) <- Now():
        default:
        }
    }

#### 实现超时机制

带超时机制的 select，是 Go 中常见的一种 select 和 channel 的组合用法

通过超时事件，我们既可以避免长期陷入某种操作的等待中，也可以做一些异常处理工作

30s 超时的 select：

    func worker() {
      select {
      case <-c:
          // ... do some stuff
      case <-time.After(30 *time.Second):
          return
      }
    }

在应用带有超时机制的 select 时，要特别注意 timer 使用后的释放，尤其在大量创建 timer 的时候

#### 实现心跳机制

结合 time 包的 Ticker，我们可以实现带有心跳机制的 select。这种机制让我们可以在监听 channel 的同时，执行一些**周期性的任务**

    func worker() {
      heartbeat := time.NewTicker(30 * time.Second) // heartbeat 实例包含一个 channel 类型的字段 C，会按一定时间间隔持续产生事件，就像“心跳”一样
      defer heartbeat.Stop()
      for {
        select {
        case <-c:
          // ... do some stuff
        case <- heartbeat.C:  // channel c 无数据接收时，会每隔特定时间完成一次迭代，然后回到 for 循环进行下一次迭代
          //... do heartbeat stuff
        }
      }
    }