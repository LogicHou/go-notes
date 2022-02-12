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