# 09 接口

**接口类型是由 type 和 interface 关键字定义的一组方法集合**，其中方法集合唯一确定了这个接口类型所表示的接口

一个典型的接口类型 MyInterface 的定义：

    type MyInterface interface {
        M1(io.Writer, ...string)            // 参数列表不需要写出形参名字
        M2(int) error                       // 返回值列表也是如此
    }

    // 和上面的写法等价
    type MyInterface interface { 
        M1(w io.Writer, args ...string)
        M2(n int) error
    }

Go 接口类型声明中的**方法必须是具名的**，并且**方法名字在这个接口类型的方法集合中是唯一的**

Go 接口类型允许嵌入的不同接口类型的方法集合存在交集，但前提是交集中的方法不仅名字要一样，它的函数签名部分也要保持一致，也就是参数列表与返回值列表也要相同，否则 Go 编译器会报错：

    type Interface1 interface {
        M1()
    }
    type Interface2 interface {
        M1(string) 
        M2()
    }

    type Interface3 interface{
        Interface1
        Interface2 // 编译器报错：duplicate method M1
        M3()
    }

所以接口的交集方法之间不存在覆盖不覆盖的情况，因为 Go 语言规定他们的方法签名必须是一致的

**在 Go 接口类型中的方法集合中可以使用首字母小写将方法设置为时非导出的**，如果包含非导出的集合方法，那么这个接口类型自身通常也是非导出的，应用范围也仅局限于包内，实际上极少使用这种带有非导出方法的接口类型，了解一下即可

## 空接口

如果一个接口类型定义中没有一个方法，那么它的方法集合就为空：

    type EmptyInterface interface {

    }

空接口可以使用 interface{} 这个类型字面值，作为所有空接口类型的代表

接口类型一旦被定义后，它就和其他 Go 类型一样可以用于声明变量：

    var err error   // err是一个error接口类型的实例变量
    var r io.Reader // r是一个io.Reader接口类型的实例变量

这些类型为接口类型的变量被称为**接口类型变量**，如果没有被显式赋予初值，接口类型变量的默认值为 **nil**

Go 规定：**如果一个类型 T 的方法集合是某接口类型 I 的方法集合的等价集合或超集，那么类型 T 就算实现了接口类型 I，那么类型 T 的变量就可以作为合法的右值赋值给接口类型 I 的变量**

由于空接口类型的方法集合为空，任何类型都实现了空接口的方法集合，所以可以将任何类型的值作为右值，赋值给空接口类型的变量：

    var i interface{} = 15 // ok
    i = "hello, golang" // ok
    type T struct{}
    var t T
    i = t  // ok
    i = &t // ok

go1.18 增加了 any 关键字，用以替代现在的 interface{} 空接口类型：type any = interface{}，实际上是 interface{} 的别名

### 类型断言

“**类型断言（Type Assertion）**”是对接口类型变量赋值的“逆操作”，也就是通过接口类型变量“还原”它的右值的类型与值信息，通常使用下面的语法形式：

    v, ok := i.(T)

如果接口类型变量 i 之前被赋予的值确为 T 类型的值，那么这个语句执行后，左侧“comma, ok”语句中的变量 ok 的值将为 true，变量 v 的类型为 T，它值会是之前变量 i 的右值

如果 i 之前被赋予的值不是 T 类型的值，那么这个语句执行后，变量 ok 的值为 false，**变量 v 的类型还是那个要还原的类型**，但它的值是类型 T 的零值

另一种形式的写法，一旦接口变量 i 之前被赋予的值不是 T 类型的值，那么这个语句将抛出 panic ，所以并不推荐使用这种方式：

    v := i.(T)

如果v, ok := i.(T)中的 T 是一个接口类型，那么类型断言的语义就会变成：断言 i 的值实现了接口类型 T

* 如果断言成功，变量 v 的类型为 i 的值的类型，而并非接口类型 T

* 如果断言失败，v 的类型信息为接口类型 T，它的值为 nil

接口类型的类型断言还有一个变种，就是 type switch，结合 switch 语句用得比较多

## 尽量定义“小接口”

接口类型的背后，是通过把类型的行为抽象成**契约**，建立双方共同遵守的约定，从而将双方的耦合降到了最低的程度

Go 选择了去繁就简的形式，主要体现在以下两点上：

* 隐式契约，无需签署，自动生效：实现者只需要实现接口方法集合中的全部方法便算是遵守了契约，并立即生效
* 更倾向于“小契约”：**尽量定义小接口，即方法个数在 1~3 个之间的接口**，“接口越大，抽象程度越弱”

### 小接口的优势

* 接口越小，抽象程度越高
* 小接口易于实现和测试
* 小接口表示的“契约”职责单一，易于复用组合

抽象程度越高，对应的集合空间就越大

抽象程度越低，越具像化，更接近事物真实面貌，对应的集合空间越小

### 定义小接口，可以遵循的几点

* 首先，别管接口大小，先抽象出接口
* 第二，将大接口拆分为小接口，可以根据哪些场合经常使用哪些方法为依据来进行拆分
* 最后，要注意接口的单一契约职责，可以考量一下现有小接口是否需要满足单一契约职责，如果需要，就可以进一步拆分，提升抽象程度

## 接口的静态特性与动态特性

接口的**静态特性**体现在**接口类型变量具有静态类型**，拥有静态类型，编译器就会在编译阶段对所有接口类型变量的赋值操作进行类型检查，检查右值的类型是否实现了该接口方法集合中的所有方法

接口的**动态特性**，体现在接口类型变量在运行时还存储了右值的真实类型信息，这个右值的真实类型被称为接口类型变量的**动态类型**

这种“动静皆备”的特性，可以带来一定的好处：

接口类型变量在程序运行时可以被赋值为不同的动态类型变量，每次赋值后，接口类型变量中存储的动态类型信息都会发生变化，比如著名的鸭子类型示例：

    type QuackableAnimal interface {
        Quack()
    }

    type Duck struct{}

    func (Duck) Quack() {
        println("duck quack!")
    }

    type Dog struct{}

    func (Dog) Quack() {
        println("dog quack!")
    }

    type Bird struct{}

    func (Bird) Quack() {
        println("bird quack!")
    }                         
                              
    func AnimalQuackInForest(a QuackableAnimal) {
        a.Quack()             
    }                         
                              
    func main() {             
        animals := []QuackableAnimal{new(Duck), new(Dog), new(Bird)}
        for _, animal := range animals {
            AnimalQuackInForest(animal)
        }  
    }

Go 接口还可以保证“动态特性”使用时的安全性，比如编译器在编译期就可以捕捉到将 int 类型变量传给 QuackableAnimal 接口类型变量这样的明显错误，决不会让这样的错误遗漏到运行时才被发现

### nil error 值 != nil 问题

以下这段示例代码并不会输出 ok ，而是输出了 “error occur: <nil>”：

    type MyError struct {
        error <--注意这里的error是个接口类型
    }

    var ErrBad = MyError{
        error: errors.New("bad things happened"),
    }

    func bad() bool {
        return false
    }

    func returnsError() error {
        var p *MyError = nil
        if bad() {
            p = &ErrBad
        }
        return p
    }

    func main() {
        err := returnsError()
        if err != nil {
            fmt.Printf("error occur: %+v\n", err)
            return
        }
        fmt.Println("ok")
    }

解决办法是把 returnsError() 里面p的类型改为 error，或者删除 p，直接 return &ErrBad 或者 nil

要想弄清楚这个问题，需要先了解一下接口类型变量的内部表示

### 接口类型变量的内部表示

接口类型变量的内部表示不像一个静态类型变量（如 int、float64）那样简单

接口类型变量有两种内部表示：eface 和 iface，这两种表示分别用于不同的接口类型变量：

    // $GOROOT/src/runtime/runtime2.go
    type eface struct {
        _type *_type
        data  unsafe.Pointer
    }

    type iface struct {
        tab  *itab
        data unsafe.Pointer
    }

* eface 用于表示没有方法的空接口（empty interface）类型变量，也就是 interface{} 类型的变量
* iface 用于表示其余拥有方法的接口 interface 类型变量

这两个结构的共同点是它们都有两个指针字段，并且第二个指针字段 (data) 的功能相同，都是指向当前赋值给该接口类型变量的动态类型变量的值

不同点在于 eface 表示的空接口类型并没有方法列表，它的第一个指针字段指向一个 _type 类型结构，这个结构为该接口类型变量的动态类型的信息，_type 类型结构的定义：

    // $GOROOT/src/runtime/type.go
    type _type struct {
        size       uintptr
        ptrdata    uintptr // size of memory prefix holding all pointers
        hash       uint32
        tflag      tflag
        align      uint8
        fieldAlign uint8
        kind       uint8
        // function for comparing objects of this type
        // (ptr to object A, ptr to object B) -> ==?
        equal func(unsafe.Pointer, unsafe.Pointer) bool
        // gcdata stores the GC type data for the garbage collector.
        // If the KindGCProg bit is set in kind, gcdata is a GC program.
        // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
        gcdata    *byte
        str       nameOff
        ptrToThis typeOff
    }

而 iface 除了要存储动态类型信息之外，还要存储接口本身的信息（接口的类型信息、方法列表信息等）以及动态类型所实现的方法的信息，因此 iface 的第一个字段指向一个itab类型结构：

    // $GOROOT/src/runtime/runtime2.go
    type itab struct {
        inter *interfacetype  <--interfacetype 结构，存储这个接口类型自身的信息
        _type *_type
        hash  uint32 // copy of _type.hash. Used for type switches.
        _     [4]byte
        fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
    }
    
    // $GOROOT/src/runtime/type.go
    type interfacetype struct {
        typ     _type        <--类型信息
        pkgpath name         <--包路径名
        mhdr    []imethod    <--接口方法集合切片
    }

eface 和 iface 中的 tab 和 _type 可以统一看作是动态类型的类型信息。Go 语言中每种类型都会有唯一的 _type 信息，无论是内置原生类型，还是自定义类型都有。Go 运行时会为程序内的全部类型建立只读的共享 _type 信息表，因此拥有相同动态类型的同类接口类型变量的 _type/tab 信息是相同的。

所以判断两个接口类型变量是否相同，只需要判断 _type/tab 是否相同，以及 data 指针指向的内存空间所存储的数据值是否相同就可以，这里要注意不是 data 指针的值相同。

### 各种接口变量内部表示信息

使用 println 函数输出各类接口类型变量的内部表示信息，并结合输出结果，解析接口类型变量的等值比较操作

在编译阶段，编译器会根据要输出的参数的类型将 println 替换为特定的函数：

    // $GOROOT/src/runtime/print.go
    func printeface(e eface) {
        print("(", e._type, ",", e.data, ")")
    }

    func printiface(i iface) {
        print("(", i.tab, ",", i.data, ")")
    }

#### 第一种：nil 接口变量

未赋初值的接口类型变量的值为 nil，这类变量也就是 nil 接口变量

无论是空接口类型还是非空接口类型变量，一旦变量值为 nil，那么它们内部表示均为(0x0,0x0)，也就是类型信息、数据值信息均为空

#### 第二种：空接口类型变量

对于空接口类型变量，只有 _type 和 data 所指数据内容一致的情况下，两个空接口类型变量之间才能划等号

#### 第三种：非空接口类型变量

非空接口类型变量的类型信息并不为空，数据指针为空，因此它与 nil（0x0,0x0）之间不能划等号。

上面的示例中从 returnsError 返回的 error 接口类型变量 err 的数据指针虽然为空，但它的类型信息（iface.tab）并不为空，而是 *MyError 对应的类型信息，这样 err 与 nil（0x0,0x0）相比自然不相等。

#### 第四种：空接口类型变量与非空接口类型变量的等值比较

空接口类型变量和非空接口类型变量内部表示的结构有所不同（第一个字段：_type vs. tab)，两者似乎一定不能相等。但 Go 在进行等值比较时，类型比较使用的是 eface 的 _type 和 iface 的 tab._type，因此当 eif 和 err 都被赋值为相同的类型并且 data 中的值也相等时，两者之间是划等号的。

## 接口类型的装箱（boxing）原理

**装箱（boxing）**是编程语言领域的一个基础概念，一般是指把一个值类型转换成引用类型

在 Go 语言中，将任意类型赋值给一个接口类型变量也是**装箱**操作，**接口类型的装箱实际就是创建一个 eface 或 iface 的过程**

装箱操作通过 runtime 包的 convT2E 和 convT2I 实现：

    // $GOROOT/src/runtime/iface.go
    func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
        ...
        ...
    }

    func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
        ...
        ...
    }

convT2E 用于将任意类型转换为一个 eface，convT2I 用于将任意类型转换为一个 iface。两个函数的实现逻辑相似，主要思路就是根据传入的类型信息（convT2E 的 _type 和 convT2I 的 tab._type）分配一块内存空间，并将 elem 指向的数据拷贝到这块内存空间中，最后传入的类型信息作为返回值结构中的类型信息，返回值结构中的数据指针（data）指向新分配的那块内存空间

经过装箱后，箱内的数据，也就是存放在新分配的内存空间中的数据与原变量便不再有关系

装箱是一个有性能损耗的操作，因此 Go 也在不断对装箱操作进行优化，包括对常见类型如整型、字符串、切片等提供系列快速转换函数：

    // $GOROOT/src/runtime/iface.go
    func convT16(val any) unsafe.Pointer     // val must be uint16-like
    func convT32(val any) unsafe.Pointer     // val must be uint32-like
    func convT64(val any) unsafe.Pointer     // val must be uint64-like
    func convTstring(val any) unsafe.Pointer // val must be a string
    func convTslice(val any) unsafe.Pointer  // val must be a slice

## Go 接口的应用模式或惯例

**在实际真正需要的时候才对程序进行抽象**，再通俗一些来讲，就是**不要为了抽象而抽象**

接口本质上是一种抽象，它的功能是解耦，所以这条原则也在告诉我们：**不要为了使用接口而使用接口**

接口的确可以实现**解耦**，但也会引入“抽象”的副作用，接口这种抽象也是有成本的，除了会造成运行效率的下降之外，也会影响代码的可读性

在多数情况下，在真实的生产项目中，接口都能给应用设计带来好处。借助接口来改善程序的设计，通过 Go 语言的“组合”设计哲学，可以让系统实现常说的高内聚和低耦合

### 一切皆组合

**组合**关注的就是如何将散落在各个包中的“零件”关联并组装到一起，是 Go 语言的重要设计哲学之一，而**正交性**则为组合哲学的落地提供了更为方便的条件

在计算机技术中，正交性用于表示某种不相依赖性或是解耦性。如果两个或更多事物中的一个发生变化，不会影响其他事物，那么这些事物就是正交的。比如，在设计良好的系统中，数据库代码与用户界面是正交的：你可以改动界面，而不影响数据库；更换数据库，而不用改动界面

**编程语言的语法元素间和语言特性也存在着正交的情况，并且通过将这些正交的特性组合起来，可以实现更为高级的特性**

Go 语言提供了诸多正交的语法元素供后续组合使用，包括：

* Go 语言无类型体系（Type Hierarchy），没有父子类的概念，类型定义是正交独立的
* 方法和类型是正交的，每种类型都可以拥有自己的方法集合，方法本质上只是一个将 receiver 参数作为第一个参数的函数而已
* 接口与它的实现者之间无“显式关联”，也就说接口与 Go 语言其他部分也是正交的

### 构建 Go 应用程序的静态骨架结构主要有两种组合方式

#### 垂直组合

垂直组合更多用在将多个类型通过“类型嵌入（Type Embedding）”的方式实现**新类型的定义**，通过这种垂直组合，可以达到方法实现的复用、接口定义重用等目的

垂直组合的几种方式：

**第一种**：通过嵌入接口构建接口，通过在接口定义中嵌入其他接口类型，实现接口行为聚合，组成大接口：

    // $GOROOT/src/io/io.go
    type ReadWriter interface {
        Reader
        Writer
    }

**第二种**：通过嵌入接口构建结构体类型，可以用于快速构建满足某一个接口的结构体类型：

    type MyReader struct {
      io.Reader // underlying reader
      N int64   // max bytes remaining
    }

**第三种**：通过嵌入结构体类型构建新结构体类型，通过这种方式构建的新结构体类型就“继承”了被嵌入的结构体的方法的实现，对新结构体类型的方法调用，可能会被“委派”给该结构体内部嵌入的结构体的实例

包括嵌入接口类型在内的各种垂直组合更多用于类型定义层面，本质上它是一种**类型组合**，也是一种类型之间的耦合方式

#### 水平组合

日常编码中，通常都可以很容易地写出下面的函数：

    func Save(f *os.File, data []byte) error

这里使用一个 *os.File 来表示数据写入的目的地，这个函数实现后可以工作得很好，但是却缺乏灵活性。os.File 是一个封装了磁盘文件描述符（又称句柄）的结构体，只有通过打开或创建真实磁盘文件才能获得这个结构体的实例，这就意味着，如果要对 Save 这个函数进行单元测试，就必须使用真实的磁盘文件而不能使用 mock 数据

Save 函数违背了接口分离（ISP）原则，也就是客户端不应该被迫依赖他们不使用的方法。os.File 不仅包含 Save 函数需要的与写数据相关的 Write 方法，还包含了其他与保存数据到文件操作不相关的方法，os.File 还包含了这些方法：

    func (f *File) Chdir() error
    func (f *File) Chmod(mode FileMode) error
    func (f *File) Chown(uid, gid int) error
    ... ...

Save 函数对 os.File 的强依赖也让它失去了扩展性。像 Save 这样的功能函数，它日后很大可能会增加向网络存储写入数据的功能需求。但如果到那时我们再来改变 Save 函数的函数签名（参数列表 + 返回值）的话，将影响到 Save 函数的所有调用者

Save 函数所在的“器官”与 os.File 所在的“器官”之间采用了一种硬连接的方式，而以 os.File 这样的结构体作为“关节”让它连接的两个“器官”丧失了相互运动的自由度，让它与它连接的两个“器官”构成的联结体变得“僵直”

而水平组合的理念就是用**接口类型参数**替换掉这些”僵直“的结构体类型，新版的 Save 函数原型如下：

    func Save(w io.Writer, data []byte) error

用 io.Writer 接口类型替换掉了 *os.File，io.Writer 仅包含一个 Save 唯一需要的 Write 方法，就符合了接口分离原则

以 io.Writer 接口类型表示数据写入的目的地，既可以支持向磁盘写入，也可以支持向网络存储写入，或者测试时 mock 数据的写入，并支持任何实现了 Write 方法的写入行为，这让 Save 函数的扩展性得到了质的提升

**接口承担了应用骨架的“关节”角色**

用接口作为“关节（连接点）”可以将各个类型水平组合（连接）在一起，通过接口的编织，整个应用程序不再是一个个孤立的“器官”（垂直组合的那些类型），而是一幅完整的、有灵活性和扩展性的静态骨架结构

### 接口应用的几种模式

通过接口进行水平组合的基本模式就是：**使用接受接口类型参数的函数或方法**

#### 基本模式

接受接口类型参数的函数或方法是水平组合的基本语法：

    func YourFuncName(param YourInterfaceType)

就这样接口类型和它的实现者之间隐式的关系却在不经意间自然而然的满足了：依赖抽象（DIP）、里氏替换原则（LSP）、接口隔离（ISP）等代码设计原则

这一水平组合的基本模式在 Go 标准库、Go 社区第三方包中有着广泛应用，其他几种模式也是从这个模式衍生的

#### 创建模式

Go 社区流传一个经验法则：接受接口，返回结构体（Accept interfaces, return structs）

    // $GOROOT/src/log/log.go
    type Logger struct {
        mu     sync.Mutex 
        prefix string     
        flag   int        
        out    io.Writer  
        buf    []byte    
    }

    func New(out io.Writer, prefix string, flag int) *Logger {
        return &Logger{
          out: out,  // 将外部传入的 io.Writer 类型接口变量赋值后，再将整个结构体返回
          prefix: prefix, 
          flag: flag
        }
    }

创建模式通过接口，在 NewXXX 函数所在包与接口的实现者所在包之间建立了一个连接，大多数包含接口类型字段的结构体的实例化，都可以使用创建模式实现

#### 包装器模式

在基本模式的基础上，当返回值的类型与参数类型相同时，通过这个函数，可以实现对输入参数的类型的包装，并在不改变被包装类型（输入参数类型）的定义的情况下，返回具备新功能特性的、实现相同接口类型的新类型：

    func YourWrapperFunc(param YourInterfaceType) YourInterfaceType

这种接口应用模式称之为**包装器模式**，也叫装饰器模式，包装器多用于对输入数据的过滤、变换等操作

Go 标准库中一个典型的包装器模式的应用：

    // $GOROOT/src/io/io.go
    func LimitReader(r Reader, n int64) Reader { return &LimitedReader{r, n} }

    type LimitedReader struct {
        R Reader // underlying reader
        N int64  // max bytes remaining
    }

    func (l *LimitedReader) Read(p []byte) (n int, err error) {
        // ... ...
    }

LimitReader 使用示例：

    func main() {
        r := strings.NewReader("hello, gopher!\n")
        lr := io.LimitReader(r, 4)
        if _, err := io.Copy(os.Stdout, lr); err != nil {
            log.Fatal(err)
        }
    }
    // 输出 hell

由于包装器模式下的包装函数（如上面的 LimitReader）的返回值类型与参数类型相同，因此可以将多个接受同一接口类型参数的包装函数组合成一条链来调用，比如：

    YourWrapperFunc1(YourWrapperFunc2(YourWrapperFunc3(...)))

#### 适配器模式

适配器模式的核心是适配器函数类型（Adapter Function Type）

适配器函数类型是一个辅助水平组合实现的“工具”类型，强调一下，**它是一个类型！**

它可以将一个满足特定函数签名的普通函数，显式转换成自身类型的实例，转换后的实例同时也是某个接口类型的实现者

经典的 http.HandlerFunc 例子：

    func greetings(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Welcome!")
    }

    func main() {
        http.ListenAndServe(":8080", http.HandlerFunc(greetings))
    }

http.HandlerFunc 是一个满足了 http.Handler 接口的 struct 类型，定义如下：

    // $GOROOT/src/net/http/server.go
    type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
    }

    type HandlerFunc func(ResponseWriter, *Request)

    func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
        f(w, r)
    }

通过 http.HandlerFunc 这个适配器函数类型，将普通函数 greetings 快速转化为满足 http.Handler 接口的类型

经过 HandlerFunc 的适配转化后，就可以将 greetings 的实例用作实参，传递给接收 http.Handler 接口的 http.ListenAndServe 函数，从而实现基于接口的组合

#### 中间件（Middleware）

中间件（Middleware）这个词的含义可大可小，**中间件就是包装模式和适配器模式结合的产物**

### 尽量避免使用空接口作为函数参数类型

**空接口不提供任何信息（The empty interface says nothing）**

Go 编译器通过解析接口定义，得到接口的名字信息以及它的方法信息，在为这个接口类型参数赋值时，编译器就会根据这些信息对实参进行检查

在函数或方法参数中使用空接口类型，就意味着没有为编译器提供关于传入实参数据的任何信息，会失去静态类型语言类型安全检查的**“保护屏障”**，需要自己检查类似的错误，并且直到运行时才能发现此类错误

所以，建议尽可能地抽象出带有一定行为契约的接口，并将它作为函数参数类型，**尽量不要使用可以“逃过”编译器类型安全检查的空接口类型（interface{}）**

标准库中以 interface{} 为参数类型的方法和函数少之甚少，不过也还是有使用 interface{} 作为参数类型的函数或方法主要有两类：

* 容器算法类，比如：container 下的 heap、list 和 ring 包、sort 包、sync.Map 等
* 格式化 / 日志类，比如：fmt 包、log 包等