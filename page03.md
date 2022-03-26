# 03 变量与作用域

## 命名

Go语言中的函数名、变量名、常量名、类型名、语句标号和包名等所有的命名，都遵循一个简单的命名规则：

* 一个名字必须以一个字母（Unicode字母）或下划线开头
* 后面可以跟任意数量的字母、数字或下划线
* 大写字母和小写字母是不同的：user 和 User 是两个不同的名字

Go语言中类似 if 和 switch 的关键字有25个；关键字不能用于自定义名字，只能在特定语法结构中使用

    break      default       func     interface   select
    case       defer         go       map         struct
    chan       else          goto     package     switch
    const      fallthrough   if       range       type
    continue   for           import   return      var

此外，还有大约 30 多个预定义的名字，比如 int 和 true 等，主要对应内建的常量、类型和函数

    内建常量: true false iota nil

    内建类型: int int8 int16 int32 int64
              uint uint8 uint16 uint32 uint64 uintptr
              float32 float64 complex128 complex64
              bool byte rune string error

    内建函数: make len cap new append copy close delete
              complex real imag
              panic recover

这些内部预先定义的名字并不是关键字，你可以在定义中重新使用它们。在一些特殊的场景中重新定义它们也是有意义的，但是也要注意避免过度而引起语义混乱

## 变量声明

与特定位置的有明确边界的内存区域绑定在一起的特定的名字，被称为**变量**

Go 是静态语言，所有变量在使用前必须先进行声明

声明的意义在于告诉编译器该变量可以操作的内存的边界信息，比如4个字节内存还是8个字节内存，又或是256个字节内存

Go 语言的变量可以分为两类：

* 一类称为包级变量 (package varible)，也就是在包级别可见的变量。如果是导出变量（大写字母开头），那么这个包级变量也可以被视为全局变量
* 另一类则是局部变量 (local varible)，也就是 Go 函数或方法体内声明的变量，仅在函数或方法体内可见


### 变量声明的一般方法

#### 带初值的语法

    var a int = 10

* var 是修饰变量声明的关键字
* a 为变量名
* int 为该变量的类型
* 10 是变量的初值
* 变量名放在了类型的前面

#### 如果没有显式为变量赋予初值，则会赋予这个类型的零值

    var a int // a的初值为int类型的零值：0

| 内置原生类型                             | 默认值（零值）       |
| ---------------------------------------- | -------------------- |
| 所有整形类型                             | 0                    |
| 浮点类型                                 | 0.0                  |
| 布尔类型                                 | FALSE                |
| 字符串类型                               | ""                   |
| 指针、接口、切片、channel、map和函数类型 | nil                  |
| 数组、结构体类复合类型                   | 各组成元素自身的零值 |

#### 变量申明块语法：

    var (
        a int = 128
        b int8 = 6
        s string = "hello"
        c rune = 'A'
        t bool = true
    )

#### 同行多个变量声明

    var a, b, c int = 5, 6, 7

#### 申明块+多个变量申明

    var (
        a, b, c int = 5, 6, 7
        c, d, e rune = 'C', 'D', 'E'
    ) 

### 变量声明的一些语法糖

#### 省略类型信息的声明 “var varName = initExpression”

    var b = 13
    var a, b, c = 12, 'A', "hello"

编译器会根据右侧变量初值自动推导出变量的类型作为初值赋值给左值

仅适用于在变量声明的同时显式赋予变量初值的情况，下面这种写法是错误的：

    var b ❌

#### 短变量声明

最简化的变量声明形式 “varName := initExpression”

    a := 12
    b := 'A'
    c := "hello"
    a, b, c := 12, 'A', "hello"

### 最佳实践

#### 包级变量的声明形式

必须带var关键字，不能使用短变量声明形式

声明聚类: 同一类的变量声明放在一个 var 变量声明块中，不同类的声明放在不同的 var 声明块中

就近原则: 尽可能在靠近第一次使用变量的位置声明这个变量，是对变量的作用域最小化的一种实现手段

#### 局部变量的声明形式

对于延迟初始化的局部变量声明，采用通用的变量声明形式

    var err error

对于声明且显式初始化的局部变量，建议使用短变量声明形式

    a := 17
    f := 3.14
    s := "hello, world!"

    // 不接受默认类型的变量
    a := int32(17)
    f := float32(3.14)
    s := []byte("hello, world!")

尽量在分支控制时使用短变量声明形式

    for i := len(s); i > 0; {
        ...
        ...
        ...
    }

## 代码块与作用域

### 代码块

显式代码块，由两个肉眼可见的且配对的大括号包裹：

    func foo() { //代码块1
        { // 代码块2 支持嵌套，嵌套在代码块1中
            { // 代码块3 嵌套在代码块2中
                { // 代码块4 嵌套在代码块3中

                }
            }
        }
    }

隐式代码块：

* 比如整个程序的最外层代码块
* Go 包都对应一个隐式包代码块
* 每个 Go 源文件都对应着一个文件代码块
* 控制语句if、for 与 switch的隐式代码块
* 最内层的隐式代码块在 switch 或 select 语句的每个 case/default 子句中

### 作用域

一个标识符(变量也是一个标识符)的作用域就是指这个标识符在被声明后可以被有效使用的源码区域

作用域是一个编译期的概念

声明于外层代码块中的标识符，其作用域包括所有内层代码块，同时适于显式代码块与隐式代码块

#### 预定义标识符

预定义标识符位于包代码块的外层，它们的作用域是范围最大的

| 预定义标识符 | 预定义标识符                                               |
| ------------ | ---------------------------------------------------------- |
| 类型         | bool、byte、complex64、complex128、error、float32、float64 |
| 类型         | int、int8、int16、int32、int64、rune、string               |
| 类型         | uint、uint8、uint16、uint32、uint64、uintptr               |
| 常量         | true、false、iota                                          |
| 零值         | nil                                                        |
| 函数         | append、cap、close、complex、copy、delete、imag、len       |
| 函数         | make、new、panic、print、println、real、recover            |

包顶层声明中的常量、类型、变量或函数（不包括方法）、导入的包名对应的标识符的作用域是包代码块

导出标识符需同时具备两个条件，这两个条件缺一不可：

1. 这个标识符声明在包代码块中，或者它是一个字段名或方法名
2. 名字第一个字符是一个大写的 Unicode 字符

函数内部声明的常量或变量对应的标识符的作用域范围开始于常量或变量声明语句的末尾，并终止于其最内部的那个包含块的末尾

#### 避免变量遮蔽的原则

变量遮蔽问题的根本原因，就是内层代码块中声明了一个与外层代码块同名且同类型的变量，内层代码块中的同名变量就会替代那个外层变量，内层变量遮蔽了外层同名变量

变量遮蔽检查插件的安装和使用：

    安装：
    $go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest

    使用：
    $go vet -vettool=$(which shadow) -strict complex.go 

这个插件并不是万能的，预定义标识符和控制语句内的短变量申明可能无法检测到

