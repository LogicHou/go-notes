# 04

## Go 的数据类型大体分为三种

* 基本数据类型 其中数值类型又用的最多
* 复合数据类型
* 接口类型

## 基本数据类型

### 广泛使用的整型

#### 平台无关整型

    有符号整形
    ├── [类型] [长度]  [取值范围]
    ├── int8  1个字节 [-128, 127]
    ├── int16 2个字节 [-32768, 32767]
    ├── int32 4个字节 [-2147483648, 2147483647]
    └── int64 8个字节 [-9223372036854775808, 9223372036854775807]
    
    无符号整形
    ├── [类型] [长度]  [取值范围]
    ├── uint8  1个字节 [0, 255]
    ├── uint16 2个字节 [0, 65535]
    ├── uint32 4个字节 [0, 4294967295]
    └── uint64 8个字节 [0, 1844674407370955165]

#### 平台相关整型

可以通过 unsafe 包提供的 SizeOf 函数来获取

    unsafe.Sizeof(var)

    平台相关整型
    ├── [类型]                  [32位长度]   [64位长度]
    ├── 默认的有符号整形 int      32位(4字节)  64位(8字节)
    ├── 默认的无符号整形 uint     32位(4字节)  64位(8字节)
    ├── 无符号整形      uintptr  大到足以存储任意一个指针的值
    └── 有移植性要求的代码，不要强依赖这些类型的长度

#### 整型的溢出问题

如果因为参与某个运算，导致结果超出了某个整型类型的值边界，就会产生整型溢出的问题

由于整型无法表示它溢出后的那个“结果”，所以出现溢出情况后，对应的整型变量的值依然会落到它的取值范围内

    var s int8 = 127
    s += 1 // 预期128，实际结果-128

    var u uint8 = 1
    u -= 2 // 预期-1，实际结果255

#### 字面值与格式化输出

    a := 53        // 十进制
    b := 0700      // 八进制，以"0"为前缀
    c1 := 0xaabbcc // 十六进制，以"0x"为前缀
    c2 := 0Xddeeff // 十六进制，以"0X"为前缀

    # go version >= 1.13
    d1 := 0b10000001 // 二进制，以"0b"为前缀
    d2 := 0B10000001 // 二进制，以"0B"为前缀
    e1 := 0o700      // 八进制，以"0o"为前缀
    e2 := 0O700      // 八进制，以"0O"为前缀
    # 使用分隔符“_”提高可读性
    a := 5_3_7   // 十进制: 537
    b := 0b_1000_0111  // 二进制位表示为10000111 
    c1 := 0_700  // 八进制: 0700
    c2 := 0o_700 // 八进制: 0700
    d1 := 0x_5c_6d // 十六进制：0x5c6d

    # 输出格式
    var a int8 = 59
    fmt.Printf("%b\n", a) //输出二进制：111011
    fmt.Printf("%d\n", a) //输出十进制：59
    fmt.Printf("%o\n", a) //输出八进制：73
    fmt.Printf("%O\n", a) //输出八进制(带0o前缀)：0o73
    fmt.Printf("%x\n", a) //输出十六进制(小写)：3b
    fmt.Printf("%X\n", a) //输出十六进制(大写)：3B

### 浮点型

采用 [IEEE 754](https://zh.wikipedia.org/wiki/IEEE_754) 标准

Go 语言提供了 float32 与 float64 两种浮点类型，分别对应 IEEE 754 中的单精度与双精度浮点数值类型(都是平台无关的)

默认值都为 0.0 ，但是占用的内存空间大小不一样

IEEE 754 规范中存储和表示一个浮点数的标准形式，在内存中的二进制表示分三个部分：符号位、阶码（即经过换算的指数）

阶码和尾数的长度决定了浮点类型可以表示的浮点数范围与精度

日常中使用双精度浮点类型（float64）的情况更多，这也是 Go 语言中浮点常量或字面值的默认类型

#### 字面值与格式化输出

直白型：

    3.1415
    .15  // 整数部分如果为0，整数部分可以省略不写
    81.80
    82. // 小数部分如果为0，小数点后的0可以省略不写

科学计数法：

    # 十进制科学计数法
    6674.28e-2 // 6674.28 * 10^(-2) = 66.742800
    .12345E+5  // 0.12345 * 10^5 = 12345.000000
    
    # 十六进制科学计数法
    0x2.p10  // 2.0 * 2^10 = 2048.000000
    0x1.Fp+0 // 1.9375 * 2^0 = 1.937500

fmt输出形式是通过 %f ：

    var f float64 = 123.45678
    fmt.Printf("%f\n", f) // 123.456780

    # 科学计数法
    fmt.Printf("%e\n", f) // 1.234568e+02
    fmt.Printf("%x\n", f) // 0x1.edd3be22e5de1p+06

### 复数类型

Go 提供两种复数类型

* complex64   实部与虚部都是 float32 类型
* complex128  实部与虚部都是 float64 类型
* 默认赋值类型为 complex128

#### 字面值

    var c = 5 + 6i
    var d = 0o123 + .12345E+5i // 83+12345i

    # 通过函数
    var c = complex(5, 6) // 5 + 6i
    var d = complex(0o123, .12345E+5) // 83+12345i

    # 通过 real 和 imag 获取一个复数的实部与虚部，返回一个浮点类型值
    var c = complex(5, 6) // 5 + 6i
    r := real(c) // 5.000000
    i := imag(c) // 6.000000

### 创建自定义的数值类型

可以通过 type 关键字基于原生数值类型来声明一个新类型

虽然新类型的数值性质与 int32 完全相同(同底层类型)，但仍是两种完全不同的类型，无法直接相互赋值

    type MyInt int32
    
    var m int = 5
    var n int32 = 6
    var a MyInt = m // 错误：在赋值中不能将m（int类型）作为MyInt类型使用
    var a MyInt = n // 错误：在赋值中不能将n（int32类型）作为MyInt类型使用

通过显式转换，让赋值操作符左右两边的操作数保持类型一致，才能进行赋值：

    var m int = 5
    var n int32 = 6
    var a MyInt = MyInt(m) // ok
    var a MyInt = MyInt(n) // ok

### 类型别名（Type Alias）

和 type 语法不同，通过类型别名语法定义的新类型与原类型完全等价，可以完全相互替代

不再需要显式转型，就可以相互赋值：

    type MyInt = int32

    var n int32 = 6
    var a MyInt = n // ok

### 字符串类型 string

Go 原生支持字符串

#### string 类型的数据是不可变的，提高了字符串的并发安全性和存储利用率

    var s string = "hello"
    s[0] = 'k'   // 错误：字符串的内容是不可改变的
    s = "gopher" // ok

#### 没有结尾’\0’，而且获取长度的时间复杂度是常数时间，消除了获取字符串长度的开销

Go 获取字符串长度是一个常数级时间复杂度，无论字符串中字符个数有多少，都可以快速得到字符串的长度值

#### 原生支持“所见即所得”的原始字符串，大大降低构造多行字符串时的心智负担

反引号原生支持构造“所见即所得”的原始字符串，原始字符串中的任意转义字符都不会起到转义的作用

    var s string = `line1
        line2
        line3`
    fmt.Println(s)

#### 对非 ASCII 字符提供原生支持，消除了源码在不同环境下显示乱码的可能

采用 Unicode 字符集，并且以 UTF-8 编码格式存储在内存当中

#### Go 字符串的组成

字节视角

Go 语言中的字符串值也是一个可空的字节序列，字节序列中的字节个数称为该字符串的长度

一个个的字节只是孤立数据，不表意

    var s = "中国人"
    fmt.Printf("the length of s = %d\n", len(s)) // 9<--这里是字节数

    for i := 0; i < len(s); i++ {
      fmt.Printf("0x%x ", s[i]) // 0xe4 0xb8 0xad 0xe5 0x9b 0xbd 0xe4 0xba 0xba
    }
    fmt.Printf("\n")

字符串视角

字符串是由一个可空的字符序列构成，表意

    var s = "中国人"
    fmt.Println("the character count in s is", utf8.RuneCountInString(s)) // 3<--这里是字符数

    for _, c := range s {
      fmt.Printf("0x%x ", c) // 0x4e2d 0x56fd 0x4eba <--某种 Unicode 字符的表示，也就是Unicode 字符集表中的码点
    }
    fmt.Printf("\n")

#### Unicode 码点

Unicode 字符集中的每个字符，都被分配了统一且唯一的字符编号

将 Unicode 字符集中的所有字符“排成一队”，字符在这个“队伍”中的位次，就是它在 Unicode 字符集中的码点

也就说，一个码点唯一对应一个字符

#### rune 类型与字符字面值

Go 使用 rune 这个类型来表示一个 Unicode 码点

本质上是 int32 类型的别名类型，与 int32 类型完全等价

    // $GOROOT/src/builtin.go
    type rune = int32

一个 rune 实例就是一个 Unicode 字符

一个 Go 字符串也可以被视为 rune 实例的集合

可以通过字符字面值来初始化一个 rune 变量

通过单引号括起的字符字面值：

    'a'  // ASCII字符
    '中' // Unicode字符集中的中文字符
    '\n' // 换行字符
    '\'' // 单引号字符

使用 Unicode 专用的转义字符\u 或\U 作为前缀：

    '\u4e2d'     // 字符：中  接两个十六进制数，两个不够则用大U表示
    '\U00004e2d' // 字符：中  可以接四个十六进制数
    '\u0027'     // 单引号字符

用整型值(rune 本质上就是一个整型数)直接作为字符字面值：

    '\x27'  // 使用十六进制表示的单引号字符
    '\047'  // 使用八进制表示的单引号字符

#### 字符串字面值

用双引号包裹即可：

    "abc\n"
    "中国人"
    "\u4e2d\u56fd\u4eba" // 中国人
    "\U00004e2d\U000056fd\U00004eba" // 中国人
    "中\u56fd\u4eba" // 中国人，不同字符字面值形式混合在一起
    "\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba" // 十六进制表示的字符串字面值：中国人

#### UTF-8 编码方案

UTF-8 编码解决的是 Unicode 码点值在计算机中如何存储和表示（位模式）的问题

UTF-8 方案使用变长度字节，对 Unicode 字符的码点进行编码

编码采用的字节数量与 Unicode 字符在码点表中的序号有关：

* 表示序号（码点）小的字符使用的字节数量少
* 表示序号（码点）大的字符使用的字节数多

UTF-8 编码使用的字节数量从 1 个到 4 个不等：

* 前 128 个与 ASCII 字符重合的码点（U+0000~U+007F）使用 1 个字节表示
* 带变音符号的拉丁文、希腊文、西里尔字母、阿拉伯文等使用 2 个字节来表示
* 东亚文字（包括汉字）使用 3 个字节表示
* 其他极少使用的语言的字符则使用 4 个字节表示

使用 Go 标准库中的 UTF-8 包，对 Unicode 字符（rune）进行编解码：

    // rune -> []byte                                                                            
    func encodeRune() {                                                                          
        var r rune = 0x4E2D                                                                      
        fmt.Printf("the unicode charactor is %c\n", r) // 中                                     
        buf := make([]byte, 3)                                                                   
        _ = utf8.EncodeRune(buf, r) // 对rune进行utf-8编码                                                           
        fmt.Printf("utf-8 representation is 0x%X\n", buf) // 0xE4B8AD                            
    }                                                                                            
                                                                                                
    // []byte -> rune                                                                            
    func decodeRune() {                                                                          
        var buf = []byte{0xE4, 0xB8, 0xAD}                                                       
        r, _ := utf8.DecodeRune(buf) // 对buf进行utf-8解码
        fmt.Printf("the unicode charactor after decoding [0xE4, 0xB8, 0xAD] is %s\n", string(r)) // 中
    }

#### Go 字符串类型的内部表示⭐⭐⭐⭐⭐

    // $GOROOT/src/reflect/value.go

    // StringHeader是一个string的运行时表示
    type StringHeader struct {
        Data uintptr <-- Data 指向一个底层数组
        Len  int <--字符串长度
    }

string 类型其实是一个“描述符”，仅由一个指向底层存储的指针和字符串的长度字段组成，本身并不真正存储字符串数据

将 string 类型通过函数 / 方法参数传入也不会带来太多的开销，因为传入的仅仅是一个“描述符”

打印底层数组的内容：

    func dumpBytesArray(arr []byte) {
        fmt.Printf("[")
        for _, b := range arr {
            fmt.Printf("%c ", b)
        }
        fmt.Printf("]\n")
    }

    func main() {
        var s = "hello"
        hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // 将string类型变量地址显式转型为reflect.StringHeader
        fmt.Printf("0x%x\n", hdr.Data) // 0x10a30e0
        p := (*[5]byte)(unsafe.Pointer(hdr.Data)) // 获取Data字段所指向的数组的指针
        dumpBytesArray((*p)[:]) // [h e l l o ]   // 输出底层数组的内容
    }

#### Go 字符串类型的常见操作

##### 下标操作

本质上等价于底层数组的下标操作

    var s = "中国人"
    fmt.Printf("0x%x\n", s[0]) // 0xe4：字符“中” utf-8编码的第一个字节
    // 注意通过下标操作，我们获取的是字符串中特定下标上的字节，而不是字符。

##### 字符迭代

通过 for 迭代，是一种字节视角的迭代，等价于对字符串底层数组的迭代

    var s = "中国人"

    for i := 0; i < len(s); i++ {
      fmt.Printf("index: %d, value: 0x%x\n", i, s[i])
    }

通过 for range 迭代，得到的是字符串中 Unicode 字符的码点值，以及该字符在字符串中的偏移值

    var s = "中国人"

    for i, v := range s {
        fmt.Printf("index: %d, value: 0x%x\n", i, v)
    }

##### 获取字符串长度

len() 获取的是字符串的字节长度

用UTF-8 包中的 RuneCountInString 函数可以获取字符串中字符个数

##### 字符串连接

原生支持通过 +/+= 操作符进行字符串连接

虽然通过 +/+= 进行字符串连接的开发体验是最好的，但连接性能就未必是最快的

    s := "Rob Pike, "
    s = s + "Robert Griesemer, "
    s += " Ken Thompson"

    fmt.Println(s) // Rob Pike, Robert Griesemer, Ken Thompson

其他字符串连接操作函数：strings.Builder、strings.Join、fmt.Sprintf 

##### 字符串比较

支持各种比较关系操作符，包括 = =、!= 、>=、<=、> 和 <

Go 采用字典序的比较策略，分别从每个字符串的起始处，开始逐个字节地对两个字符串类型变量进行比较

当两个字符串之间出现了第一个不相同的元素，比较就结束了，这两个元素的比较结果就会做为串最终的比较结果

如果出现两个字符串长度不同的情况，长度比较小的字符串会用空元素补齐，空元素比其他非空元素都小

    func main() {
            // ==
            s1 := "世界和平"
            s2 := "世界" + "和平"
            fmt.Println(s1 == s2) // true

            // !=
            s1 = "Go"
            s2 = "C"
            fmt.Println(s1 != s2) // true

            // < and <=
            s1 = "12345"
            s2 = "23456"
            fmt.Println(s1 < s2)  // true
            fmt.Println(s1 <= s2) // true

            // > and >=
            s1 = "12345"
            s2 = "123"
            fmt.Println(s1 > s2)  // true
            fmt.Println(s1 >= s2) // true
    }

##### 字符串转换

支持字符串与字节切片、字符串与 rune 切片的双向转换，使用显式类型转换就可以无需调用任何函数

string 转切片，或者切片转 string，都有一定的性能开销，因为string 是不可变的，运行时要为转换后的类型分配新内存

    var s string = "中国人"
                          
    // string -> []rune
    rs := []rune(s) 
    fmt.Printf("%x\n", rs) // [4e2d 56fd 4eba]
                    
    // string -> []byte
    bs := []byte(s) 
    fmt.Printf("%x\n", bs) // e4b8ade59bbde4baba
                    
    // []rune -> string
    s1 := string(rs)
    fmt.Println(s1) // 中国人
                    
    // []byte -> string
    s2 := string(bs)
    fmt.Println(s2) // 中国人

## 常量

* 支持无类型常量
* 支持隐式自动转型
* 可用于实现枚举

Go 常量的类型只局限于 Go 基本数据类型，包括数值类型、字符串类型，以及只有两个取值（true 和 false）的布尔类型

被创建并初始化后的常量还可以作为其他常量的初始表达式的一部分

使用const 关键字来声明常量：

    const Pi float64 = 3.14159265358979323846 // 单行常量声明

    // 以const代码块形式声明常量
    const (
        size int64 = 4096
        i, j, s = 13, 14, "bar" // 单行声明多个常量
    )

### 无类型常量

有类型常量中，即便两个类型拥有着相同的底层类型，也不能互相赋值或者进行运算，可以通过显式转型后再进行操作

如果不给常量定义类型，就会自动隐式的进行转换，并根据赋予的初值形式来决定默认类型：

    type myInt int
    const n = 13

    func main() {
        var a myInt = 5
        fmt.Println(a + n)  // 输出：18 自动将 a+n 表达式中的常量 n 转型为 myInt 类型，再与变量 a 相加
    }

### 隐式转型

对于无类型常量参与的表达式求值，Go 编译器会根据上下文中的类型信息，把无类型常量自动转换为相应的类型后，再参与求值计算

这一转型动作是隐式进行的

对常量进行转型并不会引发类型安全问题，Go 编译器会保证这一转型的安全性

如果无法将常量转换为目标类型，Go 编译器会报错：

    const m = 1333333333 // 已经超出了 int8 类型可以表示的范围，整形溢出

    var k int8 = 1
    j := k + m // 编译器报错：constant 1333333333 overflows int8

在 Go 中，使用无类型常量是一种惯用法

### 实现枚举

Go 语言并没有原生提供枚举类型，而使用 const 代码块定义的常量集合，来实现枚举

枚举类型本质上就是一个由有限数量常量所构成的集合，所以这样做并没有什么问题

Go 枚举特性：

* 自动重复上一行
* 引入 const 块中的行偏移量指示器 iota

#### 自动重复上一行

    const (
        Apple, Banana = 11, 22
        Strawberry, Grape  = 11, 22 // 使用上一行的初始化表达式
        Pear, Watermelon  = 11, 22 // 使用上一行的初始化表达式
    )

#### 偏移量指示器 iota

iota 是 Go 语言的一个预定义标识符，表示的是 const 声明块（包括单行声明）中，每个常量所处位置在块中的偏移值（从零开始）

同时，每一行中的 iota 自身也是一个无类型常量，自动参与到不同类型的求值过程中来，不需要再进行显式转型操作

位于同一行的 iota 即便出现多次，多个 iota 的值也是一样的：

    const (
        Apple, Banana = iota, iota + 10 // 0, 10 (iota = 0)
        Strawberry, Grape // 1, 11 (iota = 1)
        Pear, Watermelon  // 2, 12 (iota = 2)
    )

略过 iota = 0，从 iota = 1 开始正式定义枚举常量：

    // $GOROOT/src/syscall/net_js.go
    const (
        _ = iota
        IPV6_V6ONLY  // 1
        SOMAXCONN    // 2
        SO_ERROR     // 3
    )

想要枚举常量值并不连续，而是要略过某一个或几个值：

    const (
        _ = iota // 0
        Pin1
        Pin2
        Pin3
        _       // 使用空白标识符略过了4
        Pin5    // 5   
    )

多个 const 代码块定义的不同枚举，每个 const 代码块中的 iota 也是独立变化的：

    const (
        a = iota + 1 // 1, iota = 0
        b            // 2, iota = 1
        c            // 3, iota = 2
    )

    const (
        i = iota << 1 // 0, iota = 0
        j             // 2, iota = 1
        k             // 4, iota = 2
    )

每个 iota 的生命周期都始于一个 const 代码块的开始，在该 const 代码块结束时结束