# 08 方法

Go 方法的一般形式：

     1.关键字 2.recevier    3.方法名            4.参数列表                 5.返回值列表
        ____ _____________ __________________ ________________________  _____
        func (srv *Server) ListenAndServerTLS(certFile, keyFile string) error {
        |----------------------------|
        |  if srv.shuttingDown() {   |
        |    return ErrServerClosed  |
        |  }                         |
        |  addr := srv.Addr          |  6.方法体
        |  if addr == ""{            |
        |    addr = ":https"         |
        |  }                         |
        |  ... ...                   |
        |____________________________|
        }

Go 方法的声明比函数多出了一个** receiver 参数**的部分，**这个 receiver 参数也是方法与类型之间的纽带，也是方法与函数的最大不同**

方法中的这几个部分和函数声明中对应的部分，在形式与语义方面都是一致的，比如：方法名字首字母大小写决定该方法是否是导出方法；方法参数列表支持变长参数；方法的返回值列表也支持具名返回值等

## receiver 参数的相关约束

Go 中的方法必须是**归属于一个类型**，而 receiver 参数的类型就是这个方法归属的类型，或者说这个方法就是这个类型的一个方法

一个方法的一般声明形式：

    func (t *T或T) MethodName(参数列表) (返回值列表) {
        // 方法体
    }

无论 receiver 参数的类型为 *T 还是 T，我们都把一般声明形式中的 T 叫做 receiver 参数 t 的基类型

* 如果 t 的类型为 T，那么说这个方法是类型 T 的一个方法
* 如果 t 的类型为 *T，那么就说这个方法是类型 *T 的一个方法
* 每个方法只能有一个 receiver 参数
* 方法接收器（receiver）参数、函数 / 方法参数，以及返回值变量对应的作用域范围，都是函数 / 方法体对应的显式代码块

receiver 部分的参数名在方法作用域中需要保持唯一性：

    type T struct{}

    func (t T) M(t string) { // 编译器报错：duplicate argument t (重复声明参数t)
        ... ...
    }

如果在方法体中没有用到 receiver 参数，也可以省略 receiver 的参数名（很少用到）：

    type T struct{}

    func (T) M(t string) { 
        ... ...
    }

**receiver 参数的基类型本身不能为指针类型或接口类型**：

    type MyInt *int
    func (r MyInt) String() string { // r的基类型为MyInt，编译器报错：invalid receiver type MyInt (MyInt is a pointer type)
        return fmt.Sprintf("%d", *(*int)(r))
    }

    type MyReader io.Reader
    func (r MyReader) Read(p []byte) (int, error) { // r的基类型为MyReader，编译器报错：invalid receiver type MyReader (MyReader is an interface type)
        return r.Read(p)
    }

**方法声明要与 receiver 参数的基类型声明放在同一个包内**，因为：

* 不能为原生类型（诸如 int、float64、map 等）添加方法
* 不能跨越 Go 包为其他包的类型声明新方法

如果 receiver 参数的基类型为 T，那么 receiver 参数绑定在了 T 上，可以通过 *T 或 T 的变量实例调用该方法：

    type T struct{}

    func (t T) M(n int) {
    }

    func main() {
        var t T
        t.M(1) // 通过类型T的变量实例调用方法M

        p := &T{}
        p.M(2) // 通过类型*T的变量实例调用方法M
    }

## 方法的本质

**Go 语言中的方法的本质就是，一个以方法的 receiver 参数作为第一个参数的普通函数**

Go 中的类型是通过 receiver 联系在一起，可以为任何非内置原生类型定义方法，比如下面的类型 T：

    type T struct { 
        a int
    }

    func (t T) Get() int {  
        return t.a
    }

    func (t *T) Set(a int) int { 
        t.a = a 
        return t.a 
    }

上面示例中的类型 T 和 *T 的方法，也可以分别等价转换为下面的普通函数：

    // 类型T的方法Get的等价函数
    func Get(t T) int {  
        return t.a 
    }

    // 类型*T的方法Set的等价函数
    func Set(t *T, a int) int { 
        t.a = a 
        return t.a 
    }

调用方式的等价转换：

    // 以方法的形式调用
    var t T
    t.Get()
    (&t).Set(1)
    t.Set(1)  // 因为 receiver 参数绑定在了 T 上，所这里这样也可以

    // 直接以类型名 T 调用方法的表达方式，被称为 Method Expression
    // 通过 Method Expression 这种形式
    // 类型 T 只能调用 T 的方法集合（Method Set）中的方法
    // 同理类型 *T 也只能调用 *T 的方法集合中的方法
    var t T
    T.Get(t)
    (*T).Set(&t, 1)
    T.Set(&t, 1) // 这样则不行

方法自身的类型就是一个普通函数的类型，我们甚至可以将它作为右值，赋值给一个函数类型的变量：

    var t T
    f1 := (*T).Set // f1的类型，也是*T类型Set方法的类型：func (t *T, int)int
    f2 := T.Get    // f2的类型，也是T类型Get方法的类型：func(t T)int
    fmt.Printf("the type of f1 is %T\n", f1) // the type of f1 is func(*main.T, int) int
    fmt.Printf("the type of f2 is %T\n", f2) // the type of f2 is func(main.T) int
    f1(&t, 3)
    fmt.Println(f2(t)) // 3

方法本质上也是函数，所以关于函数设计的内容对方法也同样适用，比如错误处理设计、针对异常的处理策略、使用 defer 提升简洁性，等等

## receiver 参数类型对 Go 方法的影响

看两个 Go 方法，以及它们等价转换后的函数：

    func (t T) M1()  <=> F1(t T)
    func (t *T) M2() <=> F2(t *T)

当 receiver 参数的类型为 T 时：

Go 函数的参数采用的是值拷贝传递，所以 F1 函数体中的 t 是 **T 类型实例的一个副本**，这样在 F1 函数的实现中对参数 t 做任何修改，都只会影响副本，而不会影响到原 T 类型实例

当 receiver 参数的类型为 *T 时：

传递给 F2 函数的 t 是 T 类型实例的地址，这样 F2 函数体中对参数 t 做的任何修改，都会反映到原 T 类型实例上

## 如何选择receiver类型

### 第一个原则：

**如果 Go 方法要把对 receiver 参数代表的类型实例的修改，反映到原类型实例上，就应该选择 *T 作为 receiver 参数的类型**

    type Interface interface {
        M1()
        M2()
    }

    type T struct{}

    func (t T) M1()  {}
    func (t *T) M2() {}

    func main() {
        var t T
        var pt *T
        var i Interface

        i = pt
        i = t // cannot use t (type T) as type Interface in assignment: T does not implement Interface (M2 method has pointer receiver)
    }

T 类型的实例 t1(var t1 T) 不仅可以调用 receiver 参数类型为 T 的方法 M1，也可以调用 receiver 参数类型为 *T 的方法 M2，Go 编译器在背后会自动进行转换

t1.M2() 这种用法是 Go 提供的“语法糖”：Go 判断 t1 的类型为 T，也就是与方法 M2 的 receiver 参数类型 *T 不一致后，会自动将t1.M2()转换为(&t1).M2()

同理，类型为 \*T 的实例 t2，不仅可以调用 receiver 参数类型为 \*T 的方法 M2，还可以调用 receiver 参数类型为 T 的方法 M1，这同样是因为 Go 编译器在背后做了转换

也就是，Go 判断 t2 的类型为 \*T，与方法 M1 的 receiver 参数类型 T 不一致，就会自动将t2.M1()转换为(*t2).M1()

无论是 T 类型实例，还是 *T 类型实例，都既可以调用 receiver 为 T 类型的方法，也可以调用 receiver 为 *T 类型的方法

### 第二个原则：

根据第一原则，当需要在方法中对 receiver 参数代表的类型实例进行修改，要为 receiver 参数选择 *T 类型

如果不需要在方法中对类型实例进行修改，一般情况下，通常会为 receiver 参数选择 T 类型，因为这样可以缩窄外部修改类型实例内部状态的“接触面”，尽量少暴露可以修改类型内部状态的方法

但考虑到 Go 方法调用时，receiver 参数是以值拷贝的形式传入方法中的，**如果 receiver 参数类型的 size 较大**，以值拷贝形式传入就会导致较大的性能开销，这时选择 *T 作为 receiver 类型可能更好些

### 方法集合 - 第三个原则的预备知识

**方法集合也是用来判断一个类型是否实现了某接口类型的唯一手段，可以说，“方法集合决定了接口实现”**

Go 中任何一个类型都有属于自己的方法集合，或者说方法集合是 Go 类型的一个“属性”

但是像 int 类型就没有自己的方法，对于没有定义方法的 Go 类型，我们称其拥有空方法集合

**Go 语言规定，*T 类型的方法集合包含所有以 *T 为 receiver 参数类型的方法，以及所有以 T 为 receiver 参数类型的方法**

**而 T 类型的方法集合只包含以 T 为 receiver 参数类型的方法**

所谓的方法集合决定接口实现的含义就是：

如果某类型 T 的方法集合与某接口类型的方法集合相同，或者类型 T 的方法集合是接口类型 I 方法集合的超集，那么我们就说这个类型 T 实现了该接口

方法集合这个概念在 Go 语言中的主要用途，就是用来判断某个类型是否实现了某个接口，**如果某个类型实现了某个接口就可以把这种类型的实例值赋值给这种接口类型的变量**

示例：

    type Interface interface {
        M1()
        M2()
    }

    type T struct{}

    func (t T) M1()  {}
    func (t *T) M2() {}

    func main() {
        var t T
        var pt *T
        var i Interface

        i = pt
        i = t // cannot use t (type T) as type Interface in assignment: T does not implement Interface (M2 method has pointer receiver)
    }

### 第三个原则

**T 类型是否需要实现某个接口**，也就是是否存在将 T 类型的变量**赋值给某接口类型变量**的情况

如果 **T 类型需要实现某个接口**，那我们就要使用 T 作为 receiver 参数的类型，来满足接口类型方法集合中的所有方法

比如现在有一个接口类型I，一个自定义非接口类型T，如果希望是 *T 实现了接口 I，那么不能保证 T 也会实现 I 所以在设计一个自定义类型 T 的方法时，考虑是否 T 需要实现某个接口

如果 T 不需要实现某一接口，但 \*T 需要实现该接口，那么根据方法集合概念，\*T 的方法集合是包含 T 的方法集合的，这样我们在确定 Go 方法的 receiver 的类型时，参考原则一和原则二就可以了

如果说前面的两个原则更多聚焦于类型内部，从单个方法的实现层面考虑，那么这第三个原则则是更多从全局的设计层面考虑，聚焦于这个类型与接口类型间的耦合关系

### 最后遇事不决用指针类型就可以了

### 使用 type 定义的新类型并不会包含其基类型的方法

    type T struct{}

    func (T) M1()
    func (T) M2()
    
    type S T   // S 类型 和 *S 类型都没有包含方法，因为type S T 定义了一个新类型

    type S = T // 类型别名则可以，这里S和*S类型都包含两个方法

## 使用类型嵌入模拟实现“继承”

**独立的自定义类型**，的所有方法都是自己显式实现的，在声明类型 T 的 Go 包源码文件中一定可以找到其所有方法的实现代码

类型嵌入指的就是在一个类型的定义中嵌入了其他类型，Go 语言通过**类型嵌入（Type Embedding）**来实现“继承”

Go 语言支持两种类型嵌入，分别是**接口类型的类型嵌入**和**结构体类型的类型嵌入**

### 接口类型的类型嵌入

    type E interface {
        M1()
        M2()
    }

    type I interface {
        M1()
        M2()
        M3()
    }

    type I interface {
        E       <--嵌入了接口E，用接口类型 E 替代上面接口类型 I 定义中 M1 和 M2
        M3()
    }

这种接口类型嵌入的语义就是新接口类型（如接口类型 I）将嵌入的接口类型（如接口类型 E）的方法集合，并入到自己的方法集合中，也就是“方法集合并入”

接口类型只能嵌入接口类型

Go 语言惯例中的接口类型中只包含少量方法，并且常常只是一个方法，然后通过在接口类型中嵌入其他接口类型可以实现接口的组合，这也是 **Go 语言中基于已有接口类型构建新接口类型的惯用法**：

    // 仅包含单一方法的 io 包 Reader、Writer 和 Closer 的定义：
    // $GOROOT/src/io/io.go

    type Reader interface {
        Read(p []byte) (n int, err error)
    }

    type Writer interface {
        Write(p []byte) (n int, err error)
    }

    type Closer interface {
        Close() error
    }

    // io 包的 ReadWriter、ReadWriteCloser 等接口类型，通过嵌入上面基本接口类型组合而形成
    type ReadWriter interface {
        Reader
        Writer
    }

    type ReadCloser interface {
        Reader
        Closer
    }

    type WriteCloser interface {
        Writer
        Closer
    }

    type ReadWriteCloser interface {
        Reader
        Writer
        Closer
    }

注意这种嵌入方式在 Go 1.14 版本之前是有**约束**的：如果新接口类型嵌入了多个接口类型，这些嵌入的接口类型的方法集合不能有交集，同时嵌入的接口类型的方法集合中的方法名字，也不能与新接口中的其他方法同名：

    type Interface1 interface {
        M1()
    }

    type Interface2 interface {
        M1()
        M2()
    }
    
    // Interface1 和 Interface2 的方法集合有交集，交集是方法 M1
    type Interface3 interface {
        Interface1
        Interface2 // Error: duplicate method M1
    }

    //  Interface4 类型中的方法 M2 与嵌入的接口类型 Interface2 的方法 M2 重名
    type Interface4 interface {
        Interface2
        M2() // Error: duplicate method M2
    }

    func main() {
    }

Go 1.17 版本运行上面这个示例就不会再得到编译错误了

### 结构体类型的类型嵌入

带有嵌入字段（Embedded Field）的结构体定义：

    type T1 int
    type t2 struct{
        n int
        m int
    }

    type I interface {
        M1()
    }

    type S1 struct {
        T1
        *t2
        I            
        a int
        b string
    }

T1、*t2、I 这三个“非常规形式”的标识符**既代表字段的名字，也代表字段的类型**，它们的具体含义：

* 标识符 T1 表示字段名为 T1，它的类型为自定义类型 T1
* 标识符 t2 表示字段名为 t2，它的类型为自定义结构体类型 t2 的指针类型
* 标识符 I 表示字段名为 I，它的类型为接口类型 I

这种以某个类型名、类型的指针类型名或接口类型名，直接作为结构体字段的方式就叫做**结构体的类型嵌入**，这些字段叫做**嵌入字段（Embedded Field）**

嵌入字段的可见性与嵌入字段的类型的可见性是一致的，如果嵌入类型的名字是首字母大写的，那么也就说明这个嵌入字段是可导出的

和 Go 方法的 receiver 的基类型一样，嵌入字段类型的底层类型**不能为指针类型**

而且，嵌入字段的名字在结构体定义也必须是唯一的，这也意味着如果两个类型的名字相同，它们无法同时作为嵌入字段放到同一个结构体定义中

### 通过嵌入字段“实现继承”的原理

在调用结构体方法的时候如果发现结构体自身并没有实现该方法，Go 会查看嵌入字段对应的类型是否实现了该方法，如果有对该方法的调用就会转换成对嵌入字段对应方法的调用，比如 s.Read 的调用就被转换为 s.Reader.Read 调用

嵌入字段的方法会被提升为当前类型的方法，放入当前类型的方法集合，从外部来看，这种嵌入字段的方法的提升就有一种结构体类型“继承”了嵌入类型方法的实现

类型嵌入这种看似“继承”的机制，实际上是一种**组合的思想**，是一种组合中的代理（delegate）模式，S 只是一个代理（delegate），对外它提供了它可以代理的所有方法

### 类型嵌入与方法集合

结构体类型对嵌入类型的要求比较宽泛，可以是任意自定义类型或接口类型

#### 结构体类型中嵌入接口类型

**结构体类型的方法集合，包含嵌入的接口类型的方法集合**

注意，和接口类型中嵌入接口类型不同，当结构体嵌入的多个接口类型的方法集合**存在交集**时，Go 编译器就会因无法确定究竟使用哪个方法而报错

解决这个问题的方案有两种：

* 一是，消除方法集合存在交集的情况
* 二是，为当前类型增加交集方法的实现，这样的话，编译器便会直接选择自己实现的方法，不会陷入两难境地

嵌入某接口类型的结构体类型的方法集合，包含了这个被嵌入接口类型的方法集合，这就意味着，这个结构体类型也成为了嵌入的接口类型的一个实现，即便结构体类型自身并没有实现这个接口类型的任意一个方法，也没有关系

结构体类型嵌入接口类型在日常编码中有一个妙用，就是可以简化**单元测试的编写**，对于比如依赖外部数据库操作的方法，惯例是使用“伪对象（fake object）”来冒充真实的接口实现来进行测试

通过类型嵌入就可以快速的实现对象的接口，而不需要为伪对象去一个个实现接口的所有方法

#### 结构体类型中嵌入结构体类型

    type T1 struct{}

    func (T1) T1M1()   { println("T1's M1") }
    func (*T1) PT1M2() { println("PT1's M2") }

    type T2 struct{}

    func (T2) T2M1()   { println("T2's M1") }
    func (*T2) PT2M2() { println("PT2's M2") }

    type T struct {
        T1
        *T2
    }

* 类型 T 的方法集合 = T1 的方法集合 + *T2 的方法集合
* 类型 *T 的方法集合 = *T1 的方法集合 + *T2 的方法集合

### defined 类型与 alias 类型的方法集合

Go 语言中，凡通过类型声明语法声明的类型都被称为 defined 类型：

    type I interface {
        M1()
        M2()
    }
    type T int
    type NT T // 基于已存在的类型T创建新的defined类型NT
    type NI I // 基于已存在的接口类型I创建新defined接口类型NI

对于基于接口类型创建的 defined 的接口类型，它们的方法集合与原接口类型的方法集合是一致的

对于基于非接口类型的 defined 类型创建的非接口类型，新类型并不会“继承”原 defined 类型的任何一个方法，新 defined 类型要想实现那些接口，仍然需要重新实现接口的所有方法

对于基于类型别名（type alias）定义的新类型，无论类型别名还是原类型，输出的都是原类型的方法集合，所以无论原类型是接口类型还是非接口类型，类型别名都与原类型拥有完全相同的方法集合