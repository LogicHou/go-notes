# 07 函数

函数是唯一一种基于特定输入，实现特定任务并可返回任务执行结果的代码块（Go 语言中的方法本质上也是函数）

## 函数的声明

普通 Go 函数的构成：

	func funcName(param1 Type1, param2 Type2, ...) (return1 Type3, ...) {
        // body 函数体
    }

* 第一部分是关键字 func
* 第二部分是函数名，在同一个 Go 包中，函数名应该是唯一的，首字母大写可导出，小写表示只能在包内可见
* 第三部分是参数列表，**每个参数的参数名在前，参数类型在后**，支持变长参数(“…”符号)
* 第四部分是返回值列表，支持具名返回值(就是在返回值列表里带上返回值的名字，同时函数体中使用该名字返回值)
* 最后，放在一对大括号 {} 内的是函数体

变量声明的形式，等号后面的部分也可以看成是**函数字面值**：

    var funcVar = func(param1 Type1, param2 Type2, ...) (return1 Type3, ...) {
        // body 函数体
    }
    
函数字面值由函数类型与函数体组成，它特别像一个没有函数名的函数声明，因此我们也叫它匿名函数：

    s := T{}      // 使用复合类型字面值对结构体类型T的变量进行显式初始化
    f := func(){} // 使用变量声明形式的函数声明

这其实就是在声明一个以类型为**函数类型**的变量，函数声明中的函数名其实就是变量名，函数声明中的 func 关键字、参数列表和返回值列表共同构成了**函数类型**

参数列表与返回值列表的组合也被称为函数签名，是决定两个函数类型是否相同的决定因素，函数类型也可以看成是由 func 关键字与函数签名组合而成的

通常在表述函数类型时，会省略函数签名参数列表中的参数名，以及返回值列表中的返回值变量名：

    func(io.Writer, string, ...interface{}) (int, error)

如果两个函数类型的函数签名是相同的，即便参数列表中的参数名，以及返回值列表中的返回值变量名都是不同的，那么这两个函数类型也是相同类型

下面是是两个相同的函数类型，都是func (int, string) ([]string, error)：

    func (a int, b string) (results []string, err error)
    func (c int, d string) (sl []string, err error)

每个函数声明所定义的函数，仅仅是对应的函数类型的一个实例

## 函数参数

函数声明阶段，参数列表中的参数叫做**形式参数**（Parameter，简称形参），在函数体中，我们使用的都是形参

而在函数实际调用时传入的参数被称为**实际参数**（Argument，简称实参）

### 参数的传递方式

Go 语言中，函数参数传递采用是**值传递**的方式

所谓 “值传递”，就是将实际参数在内存中的表示**逐位拷贝**（Bitwise Copy）到形式参数中

对于像整型、数组、结构体这类类型，它们的内存表示就是它们自身的数据内容，因此当这些类型作为实参类型时，值传递拷贝的就是它们自身，传递的开销也与它们自身的大小成正比

但是像 string、切片、map 这些类型它们的内存表示对应的是它们数据内容的 “描述符”。当这些类型作为实参类型时，值传递拷贝的也是它们数据内容的 “描述符”，不包括数据内容本身，所以这些类型传递的开销是固定的，与数据内容大小无关。这种只拷贝“描述符”，不拷贝实际数据内容的拷贝过程，也被称为 “浅拷贝”

参数的传递有两个例外:

* 对于类型为接口类型的形参，Go 编译器会把传递的实参赋值给对应的接口类型形参
* 对于为变长参数的形参，Go 编译器会将零个或多个实参按一定形式转换为对应的变长形参，而**变长参数实际上是通过切片来实现的**

### 函数支持多返回值

函数返回值列表从形式上看主要有三种：

    func foo()                       // 无返回值
    func foo() error                 // 仅有一个返回值
    func foo() (int, string, error)  // 有2或2个以上返回值

### 具名返回值（Named Return Value）

    func funcName(...) (return1 Type3) {
        return1 = "Named Return Value"
        return
    }

return1 既是具名返回值

**Go 标准库以及大多数项目代码中的函数，都选择了使用普通的非具名返回值形式**

具名返回值在一些特定场景下也会得到应用：

* 比如当函数使用 defer，并在 defer 函数中修改外部函数返回值时，具名返回值可以让代码显得更优雅清晰

* 比如当函数的返回值个数较多时，每次显式使用 return 语句时都会接一长串返回值，用具名返回值可以让函数实现的可读性更好一些

## 函数是“一等公民”

    如果一门编程语言对某种语言元素的创建和使用没有限制，我们可以像对待值（value）一样对待这种语法元素，那么我们就称这种语法元素是这门编程语言的“一等公民”。拥有“一等公民”待遇的语法元素可以存储在变量中，可以作为参数传递给函数，可以在函数内部创建并可以作为返回值从函数返回


                                              -- 沃德·坎宁安 (Ward Cunningham) 对“一等公民”的解释

Go 具有“一等公民”的所有特征：

* 特征一：Go 函数可以存储在变量中
* 特征二：支持在函数内创建并通过返回值返回
  * 匿名函数
  * 闭包 -- 匿名函数如果使用了定义它的函数的局部变量，这样的匿名函数在 Go 中也被称为闭包（Closure）
* 特征三：作为参数传入函数
* 特征四：拥有自己的类型，比如 type HandlerFunc func(ResponseWriter, *Request)

### 函数“一等公民”特性的高效运用

#### 应用一：函数类型的显式转型

标准库 http 包中的 HandlerFunc 这个类型示例：

    func greeting(w http.ResponseWriter, r *http.Request) { //默认只是一个普通函数类型
        fmt.Fprintf(w, "Welcome, Gopher!\n")
    }                    

    func main() {
        http.ListenAndServe(":8080", http.HandlerFunc(greeting)) // 通过http.HandlerFunc将greeting函数进行了显示转型
    }

#### 应用二：利用闭包简化函数调用：

Go 闭包是在函数内部创建的匿名函数，这个匿名函数可以访问创建它的函数的参数与局部变量，可以利用闭包的这一特性来简化函数调用

利用闭包特性省去高频使用的乘数：

    func times(x, y int) int {
      return x * y
    }

    // 省去高频乘数的传入
    func partialTimes(x int) func(int) int {
      return func(y int) int {
        return times(x, y)
      }
    }
    
    func main() {
      timesTwo := partialTimes(2)   // 以高频乘数2为固定乘数的乘法函数
      timesThree := partialTimes(3) // 以高频乘数3为固定乘数的乘法函数
      fmt.Println(timesTwo(5))   // 10，等价于times(2, 5)
      fmt.Println(timesThree(6)) // 18，等价于times(3, 6)
    }

## 结合多返回值进行错误处理

Go 函数增加了多返回值机制，来支持错误状态与返回信息的分离

Go 语言惯用法，是使用 error 这个接口类型表示错误，并且按惯例，我们通常将 error 类型返回值放在返回值列表的末尾：

    // fmt包
    func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)

### error 类型与错误值构造

error 接口是 Go 原生内置的类型，定义如下：

    // $GOROOT/src/builtin/builtin.go
    type interface error {
        Error() string
    }

任何实现了 error 的 Error 方法的类型的实例，都可以作为错误值赋值给 error 接口变量

但是也不需要为了构造一个错误值，去自定义一个新类型来实现 error 接口，标准库中提供了两种方便 Go 开发者构造错误值的方法 errors.New 和 fmt.Errorf：

    err := errors.New("your first demo error")
    errWithCtx = fmt.Errorf("index %d is out of bounds", i)

这两种方法实际上返回的是同一个实现了 error 接口的类型的实例

未导出类型 errors.errorString 的定义：

    // $GOROOT/src/errors/errors.go

    type errorString struct {
        s string
    }

    func (e *errorString) Error() string {
        return e.s
    }

### 使用 error 类型的好处

* 统一了错误类型，提升代码可读性的同时，还更容易形成统一的错误处理策略
* 错误是值，可以对错误做 “==” 和 “!=” 的逻辑比较，函数调用者检视错误时的体验保持不变
* 易扩展，支持自定义错误上下文

### 错误处理策略

Go 语言中的错误处理，就是根据函数 / 方法返回的 error 类型变量中携带的错误值信息做决策，并选择后续代码执行路径的过程

#### 策略一：透明错误处理策略

完全不关心返回错误值携带的具体上下文信息，只要发生错误就进入唯一的错误处理执行路径：

    err := doSomething()
    if err != nil {
        // 不关心err变量底层错误值所携带的具体上下文信息
        // 执行简单错误处理逻辑并返回
        ... ...
        return err
    }
    
    func doSomething(...) error {
        ... ...
        return errors.New("some error occurred")
    }

在错误处理方不关心错误值上下文的前提下，透明错误处理策略能最大程度地减少错误处理方与错误值构造方之间的耦合关系

这也是 Go 语言中**最常见的错误处理策略**

#### 策略二：“哨兵”错误处理策略

Go 标准库采用了定义导出的“哨兵”错误值的方式，来辅助错误处理方检视错误值并做出错误处理分支的决策，比如下面的 bufio 包中定义的“哨兵错误”：

// $GOROOT/src/bufio/bufio.go
var (
    ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
    ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
    ErrBufferFull        = errors.New("bufio: buffer full")
    ErrNegativeCount     = errors.New("bufio: negative count")
)

利用哨兵错误，进行错误处理分支的决策：

    data, err := b.Peek(1)
    if err != nil {
        switch err {
        case bufio.ErrNegativeCount:
            // ... ...
            return
        case bufio.ErrBufferFull:
            // ... ...
            return
        case bufio.ErrInvalidUnreadByte:
            // ... ...
            return
        default:
            // ... ...
            return
        }
    }

基于“哨兵“的错误处理方式，使这些错误值和包的公共函数 / 方法一起成为了 API 的一部分，错误值也让使用这些值的错误处理方对它产生了依赖，一旦发布出去，开发者就要对它进行很好的维护，一旦“哨兵“发生了变动，依赖“哨兵“错误值的程序可能就会产生意想不到的错误

使用 Is 函数类似于把一个 error 类型变量与“哨兵”错误值进行比较：

    // 类似 if err == ErrOutOfBounds{ … }
    if errors.Is(err, ErrOutOfBounds) {
        // 越界的错误处理
    }

如果 error 类型变量的底层错误值是一个包装错误，errors.Is 方法会沿着该包装错误所在错误链，与链上所有被包装的错误进行比较，直至找到一个匹配的错误为止：

    var ErrSentinel = errors.New("the underlying sentinel error")

    func main() {
      err1 := fmt.Errorf("wrap sentinel: %w", ErrSentinel) // err1 包装了 ErrSentinel
      err2 := fmt.Errorf("wrap err1: %w", err1)            // err2 包装了 err1
      println(err2 == ErrSentinel)                         // 直接进行判断会返回 false
      if errors.Is(err2, ErrSentinel) {                    // 使用 Is 则会才链上进行查找比较
        println("err2 is ErrSentinel")                     // err2 is ErrSentinel 从链上找到了 ErrSentinel 错误，输出结束程序并返回
        return
      }

      println("err2 is not ErrSentinel")
    }

    // 输出
    false
    err2 is ErrSentinel

如果使用的是 Go 1.13 及后续版本，建议尽量使用 errors.Is 方法去检视某个错误值是否就是某个预期错误值，或者包装了某个特定的“哨兵”错误值

#### 策略三：错误值类型检视策略

“哨兵”错误处理策略，除了让错误处理方可以“有的放矢”的进行值比较之外，并没有提供其他有效的错误上下文信息

如果遇到错误处理方需要错误值提供更多的“错误上下文”的情况，需要通过自定义错误类型的构造错误值的方式，来提供**更多的“错误上下文”信息**

由于错误值都通过 error 接口变量统一呈现，要得到底层错误类型携带的错误上下文信息，错误处理方需要使用 Go 提供的**类型断言机制**或**类型选择机制**，即错误值类型检视策略

通过标准库的 json 包中自定义了一个 UnmarshalTypeError 的错误类型来理解实现：

    // $GOROOT/src/encoding/json/decode.go
    type UnmarshalTypeError struct {
        Value  string       
        Type   reflect.Type 
        Offset int64        
        Struct string       
        Field  string      
    }

    // $GOROOT/src/encoding/json/decode.go
    func (d *decodeState) addErrorContext(err error) error {
        if d.errorContext.Struct != nil || len(d.errorContext.FieldStack) > 0 {
            switch err := err.(type) {
            case *UnmarshalTypeError:   <--此处通过类型断言匹配到错误类型
                err.Struct = d.errorContext.Struct.Name() <--构建错误值携带上下文信息
                err.Field = strings.Join(d.errorContext.FieldStack, ".") <--构建错误值携带上下文信息
                return err
            }
        }
        return err
    }

同样一旦发布出去，开发者就要对它们进行很好的维护，也让使用这些类型进行检视的错误处理方对其产生了依赖

标准库 errors 包提供了 As 函数给错误处理方检视错误值，类似于通过类型断言判断一个 error 类型变量是否为特定的自定义错误类型：

    // 类似 if e, ok := err.(*MyError); ok { … }
    var e *MyError
    if errors.As(err, &e) {
        // 如果err类型为*MyError，变量e将被设置为对应的错误值
    }

如果 error 类型变量的动态错误值是一个包装错误，errors.As 函数会沿着该包装错误所在错误链，与链上所有被包装的错误的类型进行比较，直至找到一个匹配的错误类型：

    type MyError struct {
        e string
    }

    func (e *MyError) Error() string {
        return e.e
    }

    func main() {
        var err = &MyError{"MyError error demo"}
        err1 := fmt.Errorf("wrap err: %w", err)   // err1 包装了 err
        err2 := fmt.Errorf("wrap err1: %w", err1) // err2 包装了 err1
        var e *MyError
        if errors.As(err2, &e) {                  // 如果匹配成功，将错误值存到e中
            println("MyError is on the chain of err2")
            println(e == err)                  
            return                             
        }                                      
        println("MyError is not on the chain of err2")
    }

    // 输出
    MyError is on the chain of err2
    true

Go 1.13 及后续版本，尽量使用 errors.As 方法去检视某个错误值是否是某自定义错误类型的实例

#### 策略四：错误行为特征检视策略

前面三种除了“透明错误处理策略”，都在错误的构造方与错误处理方两者之间建立了耦合

为了降低错误处理方与错误值构造方的耦合，**将某个包中的错误类型归类，统一提取出一些公共的错误行为特征，并将这些错误行为特征放入一个公开的接口类型中**，即错误行为特征检视策略

以标准库中的net包为例：

    // $GOROOT/src/net/net.go
    type Error interface {
        error
        Timeout() bool    // 用于判断是否是超时错误
        Temporary() bool  // 用于判断是否是临时错误
    }

错误处理方只需要依赖这个公共接口，就可以检视具体错误值的错误行为特征信息，并根据这些信息做出后续错误处理分支选择的决策：
    
    // $GOROOT/src/net/http/server.go
    func (srv *Server) Serve(l net.Listener) error {
        ... ...
        for {
            rw, e := l.Accept()  // Accept 方法实际上返回的错误类型为*OpError
            if e != nil {
                select {
                case <-srv.getDoneChan():
                    return ErrServerClosed
                default:
                }
                if ne, ok := e.(net.Error); ok && ne.Temporary() { // 如果是net.Error接口错误类型，就把错误传给变量 ne
                    // 注：这里对临时性(temporary)错误进行处理
                    ... ...
                    time.Sleep(tempDelay)
                    continue
                }
                return e
            }
            ...
        }
        ... ...
    }

*OpError，它是 net 包中的一个自定义错误类型，实现了错误公共特征接口 net.Error ：

    // $GOROOT/src/net/net.go
    type OpError struct {
        ... ...
        // Err is the error that occurred during the operation.
        Err error
    }

    type temporary interface {
        Temporary() bool
    }

    func (e *OpError) Temporary() bool {
      if ne, ok := e.Err.(*os.SyscallError); ok {
          t, ok := ne.Err.(temporary)
          return ok && t.Temporary()
      }
      t, ok := e.Err.(temporary)
      return ok && t.Temporary()
    }

### 错误处理方式选择路径

**没有某种单一的错误处理策略可以适合所有项目或所有场合**

* 请尽量使用 “透明错误” 处理策略，降低错误处理方与错误值构造方之间的耦合
* 如果可以通过错误值类型的特征进行错误检视，那么请尽量使用 “错误行为特征检视策略”
* 在上述两种策略无法实施的情况下，再使用 “哨兵” 策略和 “错误值类型检视” 策略
* Go 1.13 及后续版本中，尽量用 errors.Is 和 errors.As 函数替换原先的错误检视比较语句

## 如何让函数更简洁健壮

健壮函数的几个特点：

* 无论调用者如何使用你的函数，你的函数都能以合理的方式处理调用者的任何输入，并给调用者返回预设的、清晰的错误值

* 即便你的函数发生内部异常，函数也会尽力从异常中恢复，尽可能地不让异常蔓延到整个程序

* 简洁优雅，函数的实现易读、易理解、更易维护，同时简洁也意味着统计意义上的更少的 bug

### 健壮性的“三不要”原则

* 不要相信任何外部输入的参数
  * 对所有输入的参数进行合法性的检查
  * 一旦发现问题，立即终止函数的执行，返回预设的错误值
* 不要忽略任何一个错误
  * 不能假定函数或方法一定会成功，一定要显式地检查这些调用返回的错误值
  * 一旦发现错误，要及时终止函数执行，防止错误继续传播
* 不要假定异常不会发生
  * 异常不是错误，错误是可预期的，也是经常会发生的，我们有对应的公开错误码和错误处理预案
  * 异常却是少见的、意料之外的，比如硬件异常、操作系统异常、语言运行时异常，还有更大可能是代码中潜在 bug 导致的异常
  * 根据函数的角色和使用场景，考虑是否要在函数内设置异常捕捉和恢复的环节

## Go 语言中的异常：panic

panic 指的是 Go 程序在运行时出现的一个异常情况

如果异常出现了，但没有被捕获并恢复，Go 程序的执行就会被终止，即便出现异常的位置不在主 Goroutine 中也会这样

panic 主要有两类来源：

* 一类是来自 Go 运行时
* 另一类则是 Go 开发人员通过 panic 函数主动触发的

无论是哪种，一旦 panic 被触发，后续 Go 程序的执行过程都是一样的，这个过程被 Go 语言称为 panicking：

    func foo() {
        println("call foo")
        bar()
        println("exit foo")
    }

    func bar() {
        println("call bar")
        panic("panic occurs in bar")
        zoo()
        println("exit bar")
    }

    func zoo() {
        println("call zoo")
        println("exit zoo")
    }

    func main() {
        println("call main")
        foo()
        println("exit main")
    }

输出：

    call main
    call foo
    call bar
    panic: panic occurs in bar

通过 **recover 函数** 捕捉 panic 并恢复程序正常执行秩序：

    func bar() {
        defer func() {
            if e := recover(); e != nil {
                fmt.Println("recover the panic:", e)
            }
        }()

        println("call bar")
        panic("panic occurs in bar")
        zoo()
        println("exit bar")
    }

上面的示例输出会变成：

    call main
    call foo
    call bar
    recover the panic: panic occurs in bar
    exit foo
    exit main

### 如何应对 panic

#### 第一点：评估程序对 panic 的忍受度

不同应用对异常引起的程序崩溃退出的忍受度是不一样的。比如，一个单次运行于控制台窗口中的命令行交互类程序（CLI），和一个常驻内存的后端 HTTP 服务器程序，对异常崩溃的忍受度就是不同的

#### 第二点：提示潜在 bug

我们经常在一些代码执行路径上，使用断言来表达这段执行路径上某种条件一定为真的信心

断言为真，则程序处于正确运行状态，断言为否就是出现了意料之外的问题，而这个问题很可能就是一个潜在的 bug，这时我们可以借助断言信息快速定位到问题所在

Go 语言标准库中并没有提供断言之类的辅助函数，但可以使用 panic，部分模拟断言对潜在 bug 的提示功能：

    func (d *decodeState) valueQuoted() interface{} {
        switch d.opcode {
        default:  // 出现了意料之外的问题
            panic(phasePanicMsg)

        case scanBeginArray, scanBeginObject: // 正确的分支
            ...

        case scanBeginLiteral: // 正确的分支
            v := d.literalInterface()
            ...
        }
        return unquotedValue{}
    }

在 Go 标准库中，大多数 panic 的使用都是充当类似断言的作用

### 第三点：不要混淆异常与错误

一旦在编写的 API 中，像checked exception那样使用 panic 作为正常错误处理的手段，把引发的panic当作错误，那么你就会给你的 API 使用者带去大麻烦！

因此，在 Go 中，作为 API 函数的作者，你一定不要将 panic 当作错误返回给 API 调用者

## 使用 defer 简化函数实现

defer 是 Go 语言提供的一种延迟调用机制，defer 的运作离不开函数：

* 在 Go 中，只有在函数（和方法）内部才能使用 defer
* defer 关键字后面只能接函数（或方法），这些函数被称为 deferred 函数。defer 将它们注册到其所在 Goroutine 中，用于存放 deferred 函数的栈数据结构中，这些 deferred 函数将在执行 defer 的函数退出前，按后进先出（LIFO）的顺序被程序调度执行
* 无论是执行到函数体尾部返回，还是在某个错误处理分支显式 return，又或是出现 panic，已经存储到 deferred 函数栈中的函数，都会被调度执行

### defer 使用的几个注意事项

#### 第一点：明确哪些函数可以作为 deferred 函数

对于自定义的函数或方法，defer 可以给与无条件的支持，但是对于有返回值的自定义函数或方法，返回值会在 deferred 函数被调度执行的时候被自动丢弃

Go 语言中除了自定义函数 / 方法，还有 Go 语言内置的 / 预定义的函数

其中 append、cap、len、make、new、imag 等内置函数都是不能直接作为 deferred 函数的

而 close、copy、delete、print、recover 等内置函数则可以直接被 defer 设置为 deferred 函数

对于那些不能直接作为 deferred 函数的内置函数，我们可以使用一个包裹它的匿名函数来间接满足要求：

    defer func() {
      _ = append(sl, 11)
    }()

#### 第二点：注意 defer 关键字后面表达式的求值时机

defer 关键字后面的表达式，是在将 deferred 函数注册到 deferred 函数栈的时候进行求值的：

    func foo1() {
        for i := 0; i <= 3; i++ {
            defer fmt.Println(i)
        }
    } // 输出 3 2 1 0

    func foo2() {
        for i := 0; i <= 3; i++ {
            defer func(n int) {
                fmt.Println(n)
            }(i)
        }
    } // 输出 3 2 1 0

    func foo3() {
        for i := 0; i <= 3; i++ {
            defer func() {
                fmt.Println(i)
            }()
        }
    } // 输出 4 4 4 4

    func main() {
        fmt.Println("foo1 result:")
        foo1()
        fmt.Println("\nfoo2 result:")
        foo2()
        fmt.Println("\nfoo3 result:")
        foo3()
    }

#### 第三点：知晓 defer 带来的性能损耗

在 Go 1.13 前的版本中，defer 带来的开销还是很大的，**使用 defer 的函数的执行时间是没有使用 defer 函数的 8 倍左右**

但从 Go 1.13 版本开始，Go 核心团队对 defer 性能进行了多次优化，到现在的 Go 1.17 版本，defer 的开销已经足够小，仅是不带有 defer 的函数的执行开销的 1.45 倍左右，已经达到了几乎可以忽略不计的程度，可以放心使用