# 09 接口

接口类型是由 type 和 interface 关键字定义的一组方法集合：

    type MyInterface interface {
        M1(int) error
        M2(io.Writer, ...string)
    }

Go 接口类型声明中的方法必须是具名的，并且方法名字在这个接口类型的方法集合中是唯一的

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

空接口可以使用 interface{} 这个类型字面值，作为所有空接口类型的代表

接口类型一旦被定义后，它就和其他 Go 类型一样可以用于声明变量：

    var err error   // err是一个error接口类型的实例变量
    var r io.Reader // r是一个io.Reader接口类型的实例变量

这些类型为接口类型的变量被称为**接口类型变量**，如果没有被显式赋予初值，接口类型变量的默认值为 nil

Go 规定：**如果一个类型 T 的方法集合是某接口类型 I 的方法集合的等价集合或超集，我们就说类型 T 实现了接口类型 I，那么类型 T 的变量就可以作为合法的右值赋值给接口类型 I 的变量**

由于空接口类型的方法集合为空，任何类型都实现了空接口的方法集合，所以我们可以将任何类型的值作为右值，赋值给空接口类型的变量：

    var i interface{} = 15 // ok
    i = "hello, golang" // ok
    type T struct{}
    var t T
    i = t  // ok
    i = &t // ok

“类型断言（Type Assertion）”是对接口类型变量赋值的“逆操作”，也就是通过接口类型变量“还原”它的右值的类型与值信息，通常使用下面的语法形式：

    v, ok := i.(T) 

如果接口类型变量 i 之前被赋予的值确为 T 类型的值，那么这个语句执行后，左侧“comma, ok”语句中的变量 ok 的值将为 true，变量 v 的类型为 T，它值会是之前变量 i 的右值

如果 i 之前被赋予的值不是 T 类型的值，那么这个语句执行后，变量 ok 的值为 false，变量 v 的类型还是那个要还原的类型，但它的值是类型 T 的零值

另一种形式的写法，一旦接口变量 i 之前被赋予的值不是 T 类型的值，那么这个语句将抛出 panic：

    v := i.(T)

接口类型的类型断言还有一个变种，就是 type switch

## 尽量定义“小接口”

接口类型的背后，是通过把类型的行为抽象成契约，建立双方共同遵守的约定，这种契约将双方的耦合降到了最低的程度

Go 选择了去繁就简的形式，主要体现在以下两点上：

* 隐式契约，无需签署，自动生效：实现者只需要实现接口方法集合中的全部方法便算是遵守了契约，并立即生效
* 更倾向于“小契约”：尽量定义小接口，即方法个数在 1~3 个之间的接口，“接口越大，抽象程度越弱”

### 小接口的优势

* 接口越小，抽象程度越高
* 小接口易于实现和测试
* 小接口表示的“契约”职责单一，易于复用组合

### 定义小接口，可以遵循的几点

* 首先，别管接口大小，先抽象出接口
* 第二，将大接口拆分为小接口
* 最后，要注意接口的单一契约职责

## 接口的静态特性与动态特性

接口的静态特性体现在接口类型变量具有静态类型，拥有静态类型，编译器就会在编译阶段对所有接口类型变量的赋值操作进行类型检查，检查右值的类型是否实现了该接口方法集合中的所有方法

接口的动态特性，体现在接口类型变量在运行时还存储了右值的真实类型信息，这个右值的真实类型被称为接口类型变量的动态类型

这种“动静皆备”的特性，可以带来一定的好处：

接口类型变量在程序运行时可以被赋值为不同的动态类型变量，每次赋值后，接口类型变量中存储的动态类型信息都会发生变化：

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

### nil error 值 != nil 问题

### 接口类型变量的内部表示

接口类型“动静兼备”的特性也决定了它的变量的内部表示绝不像一个静态类型变量（如 int、float64）那样简单，可以在$GOROOT/src/runtime/runtime2.go中找到接口类型变量在运行时的表示：

    // $GOROOT/src/runtime/runtime2.go
    type iface struct {
        tab  *itab
        data unsafe.Pointer
    }

    type eface struct {
        _type *_type
        data  unsafe.Pointer
    }

* eface 用于表示没有方法的空接口（empty interface）类型变量，也就是 interface{}类型的变量
* iface 用于表示其余拥有方法的接口 interface 类型变量

这两个结构的共同点是它们都有两个指针字段，并且第二个指针字段(data)的功能相同，都是指向当前赋值给该接口类型变量的动态类型变量的值

不同点在于 eface 表示的空接口类型并没有方法列表，它的第一个指针字段指向一个_type类型结构，这个结构为该接口类型变量的动态类型的信息

而 iface 除了要存储动态类型信息之外，还要存储接口本身的信息（接口的类型信息、方法列表信息等）以及动态类型所实现的方法的信息，因此 iface 的第一个字段指向一个itab类型结构

判断两个接口类型变量是否相同，只需要判断 _type/tab 是否相同，以及 data 指针指向的内存空间所存储的数据值是否相同就可以，这里要注意不是 data 指针的值相同

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

## 接口类型的装箱（boxing）原理

装箱（boxing）是编程语言领域的一个基础概念，一般是指把一个值类型转换成引用类型

在 Go 语言中，将任意类型赋值给一个接口类型变量也是装箱操作，接口类型的装箱实际就是创建一个 eface 或 iface 的过程

不过，装箱是一个有性能损耗的操作，因此 Go 也在不断对装箱操作进行优化，包括对常见类型如整型、字符串、切片等提供系列快速转换函数：

    // $GOROOT/src/runtime/iface.go
    func convT16(val any) unsafe.Pointer     // val must be uint16-like
    func convT32(val any) unsafe.Pointer     // val must be uint32-like
    func convT64(val any) unsafe.Pointer     // val must be uint64-like
    func convTstring(val any) unsafe.Pointer // val must be a string
    func convTslice(val any) unsafe.Pointer  // val must be a slice

## Go 接口的应用模式或惯例

在实际真正需要的时候才对程序进行抽象。再通俗一些来讲，就是不要为了抽象而抽象

接口本质上是一种抽象，它的功能是解耦，所以这条原则也在告诉我们：不要为了使用接口而使用接口

接口的确可以实现解耦，但也会引入“抽象”的副作用，接口这种抽象也是有成本的，除了会造成运行效率的下降之外，也会影响代码的可读性

### 一切皆组合

组合是 Go 语言的重要设计哲学之一，而正交性则为组合哲学的落地提供了更为方便的条件

在计算机技术中，正交性用于表示某种不相依赖性或是解耦性

如果两个或更多事物中的一个发生变化，不会影响其他事物，那么这些事物就是正交的

比如，在设计良好的系统中，数据库代码与用户界面是正交的：你可以改动界面，而不影响数据库；更换数据库，而不用改动界面

**编程语言的语法元素间和语言特性也存在着正交的情况，并且通过将这些正交的特性组合起来，我们可以实现更为高级的特性**

* Go 语言无类型体系（Type Hierarchy），没有父子类的概念，类型定义是正交独立的
* 方法和类型是正交的，每种类型都可以拥有自己的方法集合，方法本质上只是一个将 receiver 参数作为第一个参数的函数而已
* 接口与它的实现者之间无“显式关联”，也就说接口与 Go 语言其他部分也是正交的

### 构建 Go 应用程序的静态骨架结构有两种主要的组合方式

#### 垂直组合

垂直组合更多用在将多个类型通过“类型嵌入（Type Embedding）”的方式实现新类型的定义：

* 第一种：通过嵌入接口构建接口

* 第二种：通过嵌入接口构建结构体类型

* 第三种：通过嵌入结构体类型构建新结构体类型

#### 水平组合

接口分离原则（ISP 原则，Interface Segregation Principle），也就是客户端不应该被迫依赖他们不使用的方法

当我们通过垂直组合将一个个类型建立完毕后，就好比我们已经建立了整个应用程序骨架中的“器官”，通过接口可以将各个类型水平组合（连接）在一起

通过接口的编织，整个应用程序不再是一个个孤立的“器官”，而是一幅完整的、有灵活性和扩展性的静态骨架结构

### 接口应用的几种模式

#### 基本模式

接受接口类型参数的函数或方法是水平组合的基本语法：

    func YourFuncName(param YourInterfaceType)

#### 创建模式

接受接口，返回结构体

创建模式通过接口，在 NewXXX 函数所在包与接口的实现者所在包之间建立了一个连接，大多数包含接口类型字段的结构体的实例化，都可以使用创建模式实现

#### 包装器模式

在基本模式的基础上，当返回值的类型与参数类型相同时，通过这个函数，可以实现对输入参数的类型的包装，并在不改变被包装类型（输入参数类型）的定义的情况下，返回具备新功能特性的、实现相同接口类型的新类型

    func YourWrapperFunc(param YourInterfaceType) YourInterfaceType

这种接口应用模式我们叫它包装器模式，也叫装饰器模式，包装器多用于对输入数据的过滤、变换等操作

    // $GOROOT/src/io/io.go
    func LimitReader(r Reader, n int64) Reader { return &LimitedReader{r, n} }

    type LimitedReader struct {
        R Reader // underlying reader
        N int64  // max bytes remaining
    }

    func (l *LimitedReader) Read(p []byte) (n int, err error) {
        // ... ...
    }

由于包装器模式下的包装函数（如上面的 LimitReader）的返回值类型与参数类型相同，因此可以将多个接受同一接口类型参数的包装函数组合成一条链来调用

#### 适配器模式

适配器模式的核心是适配器函数类型（Adapter Function Type）

适配器函数类型是一个辅助水平组合实现的“工具”类型。这里我要再强调一下，它是一个类型

它可以将一个满足特定函数签名的普通函数，显式转换成自身类型的实例，转换后的实例同时也是某个接口类型的实现者

经典的 http.HandlerFunc 例子：

    func greetings(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Welcome!")
    }

    func main() {
        http.ListenAndServe(":8080", http.HandlerFunc(greetings))
    }

#### 中间件（Middleware）

中间件（Middleware）这个词的含义可大可小，实质上，这里的**中间件就是包装模式和适配器模式结合的产物**

### 尽量避免使用空接口作为函数参数类型

Go 编译器通过解析接口定义，得到接口的名字信息以及它的方法信息，在为这个接口类型参数赋值时，编译器就会根据这些信息对实参进行检查

在函数或方法参数中使用空接口类型，就意味着没有为编译器提供关于传入实参数据的任何信息，会失去静态类型语言类型安全检查的“保护屏障”，需要自己检查类似的错误，并且直到运行时才能发现此类错误

所以，建议尽可能地抽象出带有一定行为契约的接口，并将它作为函数参数类型，尽量不要使用可以“逃过”编译器类型安全检查的空接口类型（interface{}）

不过也还有使用 interface{ }作为参数类型的函数或方法主要有两类：

* 容器算法类，比如：container 下的 heap、list 和 ring 包、sort 包、sync.Map 等
* 格式化 / 日志类，比如：fmt 包、log 包等