# 08 方法

Go 方法的一般形式：

    1.关键字 2.recevier  3.方法名               4.参数列表             5.返回值列表
    ____ _____________ __________________ ________________________  _____
    func (srv *Server) ListenAndServerTLS(certFile, keyFile string) error {
      if srv.shuttingDown() {
        return ErrServerClosed
      }
      addr := srv.Addr            6.方法体
      if addr == ""{
        addr = ":https"
      }
      ... ...
    }

Go 方法的声明比函数多出了一个 receiver 部分，在此部分声明的参数，Go 称之为 receiver 参数，这个 receiver 参数也是方法与类型之间的纽带，也是方法与函数的最大不同

## receiver 参数的相关约束

Go 中的方法必须是**归属于一个类型的**，而 receiver 参数的类型就是这个方法归属的类型，或者说这个方法就是这个类型的一个方法

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

如果在方法体中没有用到 receiver 参数，也可以省略 receiver 的参数名：

    type T struct{}

    func (T) M(t string) { 
        ... ...
    }

receiver 参数的基类型本身不能为指针类型或接口类型：

    type MyInt *int
    func (r MyInt) String() string { // r的基类型为MyInt，编译器报错：invalid receiver type MyInt (MyInt is a pointer type)
        return fmt.Sprintf("%d", *(*int)(r))
    }

    type MyReader io.Reader
    func (r MyReader) Read(p []byte) (int, error) { // r的基类型为MyReader，编译器报错：invalid receiver type MyReader (MyReader is an interface type)
        return r.Read(p)
    }

**方法声明要与 receiver 参数的基类型声明放在同一个包内**，所以：

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

    // 直接以类型名 T 调用方法的表达方式，被称为 Method Expression
    var t T
    T.Get(t)
    (*T).Set(&t, 1)

方法自身的类型就是一个普通函数的类型，我们甚至可以将它作为右值，赋值给一个函数类型的变量

方法本质上就是函数，所以关于函数设计的内容对方法也同样适用，比如错误处理设计、针对异常的处理策略、使用 defer 提升简洁性，等等

## 方法集合

### receiver 参数类型对 Go 方法的影响

看两个 Go 方法，以及它们等价转换后的函数：

    func (t T) M1()  <=> F1(t T)
    func (t *T) M2() <=> F2(t *T)

当 receiver 参数的类型为 T 时：

Go 函数的参数采用的是值拷贝传递，所以 F1 函数体中的 t 是 T 类型实例的一个副本，这样在 F1 函数的实现中对参数 t 做任何修改，都只会影响副本，而不会影响到原 T 类型实例

当 receiver 参数的类型为 *T 时：

传递给 F2 函数的 t 是 T 类型实例的地址，这样 F2 函数体中对参数 t 做的任何修改，都会反映到原 T 类型实例上

## 如何选择receiver类型

### 第一个原则：

如果 Go 方法要把对 receiver 参数代表的类型实例的修改，反映到原类型实例上，那么我们应该选择 *T 作为 receiver 参数的类型

    type T struct {
        a int
    }
    
    func (t T) M1() {
        t.a = 10
    }
  
    func (t *T) M2() {
        t.a = 11
    }

T 类型的实例 t1(var t1 T) 可以调用 receiver 参数类型为 *T 的方法 M2，Go 编译器在背后自动进行转换

或者说，t1.M2() 这种用法是 Go 提供的“语法糖”：Go 判断 t1 的类型为 T，也就是与方法 M2 的 receiver 参数类型 *T 不一致后，会自动将t1.M2()转换为(&t1).M2()

同理，类型为 \*T(var t2 = &T{}) 的实例 t2，不仅可以调用 receiver 参数类型为 \*T 的方法 M2，还可以调用 receiver 参数类型为 T 的方法 M1，这同样是因为 Go 编译器在背后做了转换

也就是，Go 判断 t2 的类型为 \*T，与方法 M1 的 receiver 参数类型 T 不一致，就会自动将t2.M1()转换为(*t2).M1()

无论是 T 类型实例，还是 *T 类型实例，都既可以调用 receiver 为 T 类型的方法，也可以调用 receiver 为 *T 类型的方法

### 第二个原则：

如果我们不需要在方法中对类型实例进行修改

一般情况下，通常会为 receiver 参数选择 T 类型，因为这样可以缩窄外部修改类型实例内部状态的“接触面”，也就是尽量少暴露可以修改类型内部状态的方法

但考虑到 Go 方法调用时，receiver 参数是以值拷贝的形式传入方法中的，如果 receiver 参数类型的 size 较大，以值拷贝形式传入就会导致较大的性能开销，这时我们选择 *T 作为 receiver 类型可能更好些

### 方法集合 - 第三个原则的预备知识

方法集合也是用来判断一个类型是否实现了某接口类型的唯一手段，可以说，“方法集合决定了接口实现”

Go 中任何一个类型都有属于自己的方法集合，或者说方法集合是 Go 类型的一个“属性”

但是像 int 类型就没有自己的方法，对于没有定义方法的 Go 类型，我们称其拥有空方法集合

Go 语言规定，*T 类型的方法集合包含所有以 *T 为 receiver 参数类型的方法，以及所有以 T 为 receiver 参数类型的方法

而 T 类型的方法集合只包含以 T 为 receiver 参数类型的方法

所谓的方法集合决定接口实现的含义就是：

如果某类型 T 的方法集合与某接口类型的方法集合相同，或者类型 T 的方法集合是接口类型 I 方法集合的超集，那么我们就说这个类型 T 实现了接口 I

方法集合这个概念在 Go 语言中的主要用途，就是用来判断某个类型是否实现了某个接口

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

T 类型是否需要实现某个接口，也就是是否存在将 T 类型的变量赋值给某接口类型变量的情况

如果 T 类型需要实现某个接口，那我们就要使用 T 作为 receiver 参数的类型，来满足接口类型方法集合中的所有方法

如果 T 不需要实现某一接口，但 *T 需要实现该接口，那么根据方法集合概念，*T 的方法集合是包含 T 的方法集合的，这样我们在确定 Go 方法的 receiver 的类型时，参考原则一和原则二就可以了

如果说前面的两个原则更多聚焦于类型内部，从单个方法的实现层面考虑，那么这第三个原则则是更多从全局的设计层面考虑，聚焦于这个类型与接口类型间的耦合关系