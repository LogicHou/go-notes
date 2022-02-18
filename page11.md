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

**从一个已关闭的 channel 接收数据将永远不会被阻塞**

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

对同一个无缓冲 channel，只有对它进行接收操作的 Goroutine 和对它进行发送操作的 Goroutine 都存在的情况下，通信才能得以进行，否则单方面的操作会让对应的 Goroutine 陷入挂起状态：

    func main() {
        ch1 := make(chan int)
        ch1 <- 13 // fatal error: all Goroutines are asleep - deadlock!
        n := <-ch1
        println(n)
    }

改成这样即可：

    func main() {
        ch1 := make(chan int)
        go func() {
            ch1 <- 13 // 将发送操作放入一个新Goroutine中执行
        }()
        n := <-ch1 // 这里阻塞住，等待从上面的 Goroutine 中接收数据
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

简单的概括就是，满了发送会阻塞，空了接收会阻塞

示例：

    ch2 := make(chan int, 1)
    n := <-ch2 // 由于此时ch2的缓冲区中无数据，因此对其进行接收操作将导致Goroutine挂起

    ch3 := make(chan int, 1)
    ch3 <- 17  // 向ch3发送一个整型数17
    ch3 <- 27  // 由于此时ch3中缓冲区已满，再向ch3发送数据也将导致Goroutine挂起

### 关闭 channel

channel 关闭后，所有等待从这个 channel 接收数据的操作都将返回，也就是解除阻塞可以继续向下执行代码

采用不同接收语法形式的语句，在 channel 被关闭后返回值的情况：

    n := <- ch      // 当ch被关闭后，n将被赋值为ch元素类型的零值
    m, ok := <-ch   // 当ch被关闭后，m将被赋值为ch元素类型的零值, ok值为false
    for v := range ch { // 当ch被关闭后，for range循环结束，有数据会接收完数据
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

再次提及，从一个已关闭的 channel 接收数据将永远不会被阻塞

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

当 select 语句中没有 default 分支，而且所有 case 中的 channel 操作都阻塞了的时候，整个 select 语句都将被阻塞

直到某一个 case 上的 channel 变成可发送，或者某个 case 上的 channel 变成可接收，select 语句才可以继续进行下去

### 无缓冲 channel 的惯用法

无缓冲 channel 兼具通信和同步特性，在并发程序中应用颇为广泛

#### 第一种用法：用作信号传递

无缓冲 channel 用作信号传递的时候，有两种情况，分别是 1 对 1 通知信号和 1 对 n 通知信号

1 对 1 的情况下，可以通过在一个通道内返回一个专门用作通知 main Goroutine 的信号来结束阻塞动作，示例：

    type signal struct{}

    func main() {
        c := make(chan signal)
        println("start a worker...")
        go func() {
            println("worker start to work...")
            func() {
                time.Sleep(3 * time.Second)
            }()
            c <- signal{}
        }()
        <-c // 这里阻塞住了 main Goroutine 直到接收到信号才继续向下执行
        fmt.Println("worker work done!")
    }

1 对 n 的情况下，这样的信号通知机制，常被用于协调多个 Goroutine 一起工作

比如在多个 Goroutine 中植入信号接收操作阻塞住各个 Goroutine 的执行，然后通过在 main Goroutine 中 close 通道解除阻塞是 Goroutine 中的逻辑继续执行：

    type signal struct{}

    func main() {
        fmt.Println("start a group of workers...")
        groupSignal := make(chan signal)
        c := make(chan struct{}) // 负责通知 main Goroutine 退出的无缓冲 channel
        var wg sync.WaitGroup

        for i := 0; i < 2; i++ {
            wg.Add(1)
            go func(i int) {
              <-groupSignal
              fmt.Printf("worker %d: start to work...\n", i)
              time.Sleep(2 * time.Second) // do something
              fmt.Printf("worker %d: works done\n", i)
              wg.Done()
            }(i + 1)
        }

        go func() {
            wg.Wait()
            c <- signal(struct{}{})
        }()

        time.Sleep(5 * time.Second)
        fmt.Println("the group of workers start to work...")
        close(groupSignal)
        <-c
        fmt.Println("the group of workers work done!")
    }

#### 第二种用法：用于替代锁机制

无缓冲 channel 具有同步特性，这让它在某些场合可以替代锁，让我们的程序更加清晰，可读性也更好

比如可以将并发的计数器操作全部交给一个独立的 Goroutine 去处理，并通过无缓冲 channel 的同步阻塞特性，实现了计数器的控制

这样其他 Goroutine 通过增加计数器值的动作，实质上就转化为了一次无缓冲 channel 的接收动作

    type counter struct {
      c chan int
      i int
    }

    func main() {
      cter := &counter{
        c: make(chan int),
      }
      go func() {
        for {
          cter.i++
          cter.c <- cter.i // 如果不接收就会阻塞住，这里用的时无缓冲 channel ，不能同时操作，实现了锁机制
        }
      }()
      var wg sync.WaitGroup
      for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
          v := <-cter.c // 通过接收才能进行下一次累加操作
          fmt.Printf("Goroutine-%d: current counter value is %d\n", i, v)
          wg.Done()
        }(i)
      }
      wg.Wait()
    }

这种并发设计逻辑更符合 Go 语言所倡导的“**不要通过共享内存来通信，而是通过通信来共享内存**”的原则

### 带缓冲 channel 的惯用法

带缓冲的 channel 与无缓冲的 channel 的最大不同之处，就在于它的**异步性**

对于一个带缓冲 channel，在缓冲区未满的情况下，对它进行发送操作的 Goroutine 不会阻塞挂起

在缓冲区有数据的情况下，对它进行接收操作的 Goroutine 也不会阻塞挂起

#### 第一种用法：用作消息队列

和无缓冲 channel 更多用于信号 / 事件管道相比，可自行设置容量、异步收发的带缓冲 channel 更适合被用作为消息队列，并且，带缓冲 channel 在数据收发的性能上要明显好于无缓冲 channel

不过 Go 支持 channel 的初衷是作为 Goroutine 间的通信手段，并不是专门用于消息队列场景的

如果项目需要专业消息队列的功能特性，比如支持优先级、支持权重、支持离线持久化等，那么 channel 就不合适了，可以使用第三方的专业的消息队列实现

#### 第二种用法：用作计数信号量（counting semaphore）

带缓冲 channel 中的当前数据个数代表的是(一种假设)，当前同时处于活动状态（处理业务）的 Goroutine 的数量，而带缓冲 channel 的容量（capacity），就代表了允许同时处于活动状态的 Goroutine 的最大数量

向带缓冲 channel 的一个发送操作表示获取一个信号量，而从 channel 的一个接收操作则表示释放一个信号量

#### 第三种用法：实现锁机制

    var cs = 0 // 模拟临界区要保护的数据
    var c = make(chan struct{}, 1)

    func plus() {
        c <- struct{}{}
        cs++
        <-c
    }

### len(channel) 的应用

len 是 Go 语言的一个内置函数，支持接收数组、切片、map、字符串和 channel 类型的参数，并返回对应类型的“长度”，也就是一个整型值

针对 channel ch 的类型不同，len(ch) 有如下两种语义：

* 当 ch 为无缓冲 channel 时，len(ch) 总是返回 0，这里调用 len(ch) 就没什么意义
* 当 ch 为带缓冲 channel 时，len(ch) 返回当前 channel ch 中**尚未被读取**的元素个数

对带缓冲的 channel 不能直接使用 len 函数来实现“判满”、“判有”和“判空”逻辑：

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

所以单纯地依靠 len(channel) 来判断 channel 中元素状态，是不能保证在后续对 channel 的收发时 channel 状态是不变的

因此，常见的方法是将 “判空与读取” 放在一个 “事务” 中，将 “判满与写入” 放在一个 “事务” 中，而这类 “事务” 可以通过 select 实现：

    func tryRecv(c <-chan int) (int, bool) {
        select {
        case i := <-c:
            return i, true
        default:
            return 0, false
        }
    }

    func trySend(c chan<- int, i int) bool {
        select {
        case c <- i:
            return true
        default:
            return false
        }
    }

这种方法适用于大多数场合，但是这种方法有一个 “问题”，那就是它改变了 channel 的状态，会让 channel 接收了一个元素或发送一个元素到 channel

如果想要单纯地侦测 channel 的状态，而又不会因 channel 满或空阻塞在 channel 上，目前没有一种方法可以在实现这样的功能的同时，适用于所有场合

但是在特定的场景下，可以用 len(channel) 来实现，比如下面这两种场景：

是一个 “多发送单接收” 的场景，也就是有多个发送者，但有且只有一个接收者，这时可以在接收 Goroutine 中使用 len(channel) 是否大于 0 来判断是否 channel 中有数据需要接收

    for {
      if len(c) > 0 {
        i := <-c // 处理i
      }
      ......
    }

是一个 “多接收单发送” 的场景，也就是有多个接收者，但有且只有一个发送者，这时可以在发送 Goroutine 中使用 len(channel) 是否小于 cap(channel) 来判断是否可以执行向 channel 的发送操作

    for {
      if len(c) < cap(c) {
        c <- i
      }
      ......
    }

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

在应用带有超时机制的 select 时，要特别注意 **timer 使用后的释放**，尤其在大量创建 timer 的时候

作为 time.Timer 的使用者，要尽量减少在使用 Timer 时给 Go 运行时和 Go 垃圾回收带来的压力，要及时调用 timer 的 Stop 方法回收 Timer 资源

#### 实现心跳机制

结合 time 包的 Ticker，我们可以实现带有心跳机制的 select。这种机制让我们可以在监听 channel 的同时，执行一些**周期性的任务**：

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

和 timer 一样，在使用完 ticker 之后，也不要忘记调用它的 Stop 方法，避免心跳事件在 ticker 的 channel（上面示例中的 heartbeat.C）中持续产生

## 共享变量

标准库的 sync 包，提供了针对传统的、基于共享内存并发模型的低级同步原语，包括：互斥锁（sync.Mutex）、读写锁（sync.RWMutex）、条件变量（sync.Cond）等，并通过 atomic 包提供了原子操作原语等等

一般情况下，建议优先使用 CSP 并发模型进行并发程序设计，但是在一些场景中，依然需要 sync 包提供的低级同步原语

### 首先是需要高性能的临界区（critical section）同步机制场景

在 Go 中，channel 并发原语也可以用于对数据对象访问的同步，可以把 channel 看成是一种高级的同步原语，它自身的实现也是建构在低级同步原语之上的

也正因为如此，channel 自身的性能与低级同步原语相比要略微逊色，开销要更大

无论是在单 Goroutine 情况下，还是在并发测试情况下，sync.Mutex实现的同步机制的性能，都要比 channel 实现的高出很多

### 第二种就是在不想转移结构体对象所有权，但又要保证结构体内部状态数据的同步访问的场景

基于 channel 的并发设计有一个特点：在 Goroutine 间通过 channel 转移数据对象的所有权，只有拥有数据对象所有权（从 channel 接收到该数据）的 Goroutine 才可以对该数据对象进行状态变更

如果设计中没有转移结构体对象所有权，但又要保证结构体内部状态数据在多个 Goroutine 之间同步访问，那么可以使用 sync 包提供的低级同步原语来实现，比如最常用的sync.Mutex

### sync 包中同步原语使用的注意事项

不应复制那些包含了此包中类型的值

首次使用 Mutex 等 sync 包中定义的结构类型后，不应该再对它们进行复制操作

sync.Mutex 的定义非常简单，由两个整型字段 state 和 sema 组成：

    // $GOROOT/src/sync/mutex.go
    type Mutex struct {
        state int32
        sema  uint32
    }

* state：表示当前互斥锁的状态
* sema：用于控制锁状态的信号量

初始情况下，Mutex 的实例处于 Unlocked 状态（state 和 sema 均为 0）

对 Mutex 实例的复制也就是两个整型字段的复制

一旦发生复制，原变量与副本就是两个单独的内存块，各自发挥同步作用，互相就没有了关联

如果发生复制后，原变量与副本保护的不是同一个数据对象

在使用 sync 包中的类型的时候，推荐通过闭包方式，或者是传递类型实例（或包裹该类型的类型实例）的地址（指针）的方式进行

### 互斥锁（Mutex）还是读写锁（RWMutex）

    var mu sync.Mutex
    mu.Lock()   // 加锁
    doSomething()
    mu.Unlock() // 解锁

一旦某个 Goroutine 调用的 Mutex 执行 Lock 操作成功，它将成功持有这把互斥锁

这个时候，如果有其他 Goroutine 执行 Lock 操作，就会阻塞在这把互斥锁上，直到持有这把锁的 Goroutine 调用 Unlock 释放掉这把锁后，才会抢到这把锁的持有权并进入临界区

使用互斥锁的两个原则：

* **尽量减少在锁中的操作**。这可以减少其他因 Goroutine 阻塞而带来的损耗与延迟
* **一定要记得调用 Unlock 解锁**。忘记解锁会导致程序局部死锁，甚至是整个程序死锁，会导致严重的后果。也可以结合 defer，优雅地执行解锁操作。

读写锁与互斥锁用法大致相同，只不过多了一组加读锁和解读锁的方法：

    var rwmu sync.RWMutex
    rwmu.RLock()   //加读锁
    readSomething()
    rwmu.RUnlock() //解读锁
    rwmu.Lock()    //加写锁
    changeSomething()
    rwmu.Unlock()  //解写锁

写锁与 Mutex 的行为十分类似，一旦某 Goroutine 持有写锁，其他 Goroutine 无论是尝试加读锁，还是加写锁，都会被阻塞在写锁上

但读锁就宽松多了，一旦某个 Goroutine 持有读锁，它不会阻塞其他尝试加读锁的 Goroutine，但加写锁的 Goroutine 依然会被阻塞住

通常，互斥锁（Mutex）是临时区同步原语的首选，常被用来对结构体对象的内部状态、缓存等进行保护，是使用最为广泛的临界区同步原语

相比之下，读写锁的应用就没那么广泛了，只活跃于它擅长的场景下

Mutex、RWMutex 在并发下的读写性能表现：

* 并发量较小的情况下，Mutex 性能最好；随着并发量增大，Mutex 的竞争激烈，导致加锁和解锁性能下降
* RWMutex 的读锁性能并不会随着并发量的增大，而发生较大变化，性能始终比较恒定
* 在并发量较大的情况下，RWMutex 的写锁性能和 Mutex、RWMutex 读锁相比，是最差的，并且随着并发量增大，RWMutex 写锁性能有继续下降趋势

**读写锁适合应用在具有一定并发量且读多写少的场合**

在大量并发读的情况下，多个 Goroutine 可以同时持有读锁，从而减少在锁竞争中等待的时间

而互斥锁，即便是读请求的场合，同一时刻也只能有一个 Goroutine 持有锁，其他 Goroutine 只能阻塞在加锁操作上等待被调度

### 条件变量 sync.Cond

可以把一个条件变量理解为一个容器，这个容器中存放着一个或一组等待着某个条件成立的 Goroutine

当条件成立后，这些处于等待状态的 Goroutine 将得到通知，并被唤醒继续进行后续的工作

条件变量是同步原语的一种，如果没有条件变量，开发人员可能需要在 Goroutine 中通过连续轮询的方式，检查某条件是否为真，这种连续轮询非常消耗资源，因为 Goroutine 在这个过程中是处于活动状态的，但它的工作又没有进展

而 sync.Cond 为 Goroutine 在这个场景下提供了另一种可选的、资源消耗更小、使用体验更佳的同步方式

和sync.Mutex 、sync.RWMutex等相比，sync.Cond 应用的场景更为有限，只有在需要“等待某个条件成立”的场景下，Cond 才有用武之地

### 原子操作（atomic operations）

atomic 包是 Go 语言给用户提供的原子操作原语的相关接口

原子操作（atomic operations）是相对于普通指令操作而言的

    var a int
    a++

a++ 这行语句需要 3 条普通机器指令来完成变量 a 的自增：

* LOAD：将变量从内存加载到 CPU 寄存器
* ADD：执行加法指令
* STORE：将结果存储回原内存地址中

这 3 条普通指令在执行过程中是可以被中断的

而原子操作的指令是不可中断的，它就好比一个事务，要么不执行，一旦执行就一次性全部执行完毕，中间不可分割

也正因为如此，原子操作也可以被用于共享数据的并发同步

原子操作由底层硬件直接提供支持，是一种硬件实现的指令级的“事务”，相对于操作系统层面和 Go 运行时层面提供的同步技术而言，它更为原始

atomic 包封装了 CPU 实现的部分原子操作指令，为用户层提供体验良好的原子操作函数

atomic 包中提供的原语更接近硬件底层，也更为低级，也常被用于实现更为高级的并发同步技术，比如 channel 和 sync 包中的同步原语

atomic 包提供了两大类原子操作接口，一类是针对整型变量的，包括有符号整型、无符号整型以及对应的指针类型；另外一类是针对自定义类型的

atomic 原子操作的特性：随着并发量提升，使用 atomic 实现的共享变量的并发读写性能表现更为稳定，尤其是原子读操作，和 sync 包中的读写锁原语比起来，atomic 表现出了更好的伸缩性和高性能

atomic 包更适合一些对性能十分敏感、并发量较大且读多写少的场合

atomic 原子操作可用来同步的范围有比较大限制，只能同步一个整型变量或自定义类型变量。如果要对一个复杂的临界区数据进行同步，那么首选的依旧是 sync 包中的原语