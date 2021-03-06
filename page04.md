# 04 基本数据类型、常量

Go 的数据类型大体可以分为三种

* 基本数据类型 <-- 日常 Go 编码中使用最多的类型，而其中数值类型又用的最多
  * 整数类型
  * 浮点类型
  * 复数类型
  * 布尔类型
  * 字符串类型
  * byte 和 rune 类型
* 复合数据类型
  * slice
  * map
* 接口类型

# 基本数据类型

## 被广泛使用的整型

默认值都为 0

自动推导类型为平台相关的 int 类型

### 平台无关整型

有符号整形

    ├── [类型] [长度]  [取值范围]
    ├── int8  1个字节 [-128, 127]
    ├── int16 2个字节 [-32768, 32767]
    ├── int32 4个字节 [-2147483648, 2147483647]
    └── int64 8个字节 [-9223372036854775808, 9223372036854775807]
    
对于负数整数的编码，Go 采用 2 的补码作为整型的比特位编码方法

所以不能简单地将最高比特位看成负号，把其余比特位表示的值看成负号后面的数值

Go 的补码是通过原码逐位取反后再加 1 得到的：

    0 1 1 1 1 1 1 1 1   <--  127的源码
    1 0 0 0 0 0 0 0 0   <--  取反
    1 0 0 0 0 0 0 0 1   <--  +1 后就是 -127（补码表示）


无符号整形

    ├── [类型] [长度]  [取值范围]
    ├── uint8  1个字节 [0, 255]
    ├── uint16 2个字节 [0, 65535]
    ├── uint32 4个字节 [0, 4294967295]
    └── uint64 8个字节 [0, 1844674407370955165]

### 平台相关整型

    ├── [类型]                  [32位长度]   [64位长度]
    ├── 默认的有符号整形 int      32位(4字节)  64位(8字节)
    ├── 默认的无符号整形 uint     32位(4字节)  64位(8字节)
    ├── 无符号整形      uintptr  大到足以存储任意一个指针的值
    └── 编写有移植性要求的代码，不要强依赖这些类型的长度

此类型长度会根据运行平台的改变而改变

可以通过 unsafe 包提供的 SizeOf 函数来获取其长度

    unsafe.Sizeof(varName)

### 整型的溢出问题

如果因为参与某个运算，导致结果超出了某个整型类型的值边界，就会产生整型溢出的问题

由于整型无法表示它溢出后的那个“结果”，所以出现溢出情况后，对应的整型变量的值依然会落到它的取值范围内

    var s int8 = 127
    s += 1 // 预期128，实际结果-128

    var u uint8 = 1
    u -= 2 // 预期-1，实际结果255

### 字面值与格式化输出

十进制、八进制、十六进制的数值字面值形式：

    a := 53        // 十进制
    b := 0700      // 八进制，以"0"为前缀
    c1 := 0xaabbcc // 十六进制，以"0x"为前缀
    c2 := 0Xddeeff // 十六进制，以"0X"为前缀

Go 1.13 版本开始，Go 又增加了对二进制字面值的支持和两种八进制字面值的形式：

    d1 := 0b10000001 // 二进制，以"0b"为前缀
    d2 := 0B10000001 // 二进制，以"0B"为前缀
    e1 := 0o700      // 八进制，以"0o"为前缀
    e2 := 0O700      // 八进制，以"0O"为前缀

使用分隔符 “_” 来将数字分组以提高可读性：

    a := 5_3_7   // 十进制: 537
    b := 0b_1000_0111  // 二进制位表示为10000111 
    c1 := 0_700  // 八进制: 0700
    c2 := 0o_700 // 八进制: 0700
    d1 := 0x_5c_6d // 十六进制：0x5c6d

可以通过标准库 fmt 包的格式化输出函数，将一个整型变量输出为不同进制的形式：

    var a int8 = 59
    fmt.Printf("%b\n", a) //输出二进制：111011
    fmt.Printf("%d\n", a) //输出十进制：59
    fmt.Printf("%o\n", a) //输出八进制：73
    fmt.Printf("%O\n", a) //输出八进制(带0o前缀)：0o73
    fmt.Printf("%x\n", a) //输出十六进制(小写)：3b
    fmt.Printf("%X\n", a) //输出十六进制(大写)：3B

## 浮点类型

Go 的浮点型采用 [IEEE 754](https://zh.wikipedia.org/wiki/IEEE_754) 二进制浮点数算术标准

Go 语言提供了 **float32** 与 **float64** 两种浮点类型，分别对应 IEEE 754 中的单精度与双精度浮点数值类型(都是平台无关的)

两种浮点类型的默认值都为 0.0 ，但是占用的内存空间大小不一样

IEEE 754 规范在内存中存储和表示一个浮点数的标准形式，内存中的二进制表示分三个部分：**符号位、阶码（即经过换算的指数），以及尾数**

@todo 详细运算过程

日常中使用双精度浮点类型（float64）的情况更多，这也是 Go 语言中浮点常量或字面值的默认类型

自动推导类型为 float64

float32 由于表示范围与精度有限，可能会导致输出结果与常识不符。比如两个浮点类型变量被两个不同的浮点字面值初始化，但逻辑比较的结果却是两个变量的值相等：

    var f1 float32 = 16777216.0
    var f2 float32 = 16777217.0
    fmt.Println(f1 == f2) // true

这时因为它们转换成二进制的数据是相等的，所以要小心使用浮点类型数值比较后的结果，开发中如果能避开就避开，例如金钱单位只有美元或者人民币建议以分作为单位

### 字面值与格式化输出

直白地用十进制表示的浮点值形式：

    3.1415
    .15  // 整数部分如果为0，整数部分可以省略不写
    81.80
    82. // 小数部分如果为0，小数点后的0可以省略不写

科学计数法形式：

    # 十进制科学计数法，e/E 代表的幂运算的底数为 10
    6674.28e-2 // 6674.28 * 10^(-2) = 66.742800
    .12345E+5  // 0.12345 * 10^5 = 12345.000000
    
    # 十六进制科学计数法，整数部分、小数部分用的都是十六进制形式，指数部分依然是十进制形式，p/P 代表的幂运算的底数为 2
    0x2.p10  // 2.0 * 2^10 = 2048.000000
    0x1.Fp+0 // 1.9375 * 2^0 = 1.937500

fmt 包提供了针对浮点数的格式化输出形式是 %f ：

    var f float64 = 123.45678
    fmt.Printf("%f\n", f) // 123.456780

    # 科学计数法形式，%e 输出的是十进制的科学计数法形式，%x 输出的是十六进制的科学计数法形式
    fmt.Printf("%e\n", f) // 1.234568e+02
    fmt.Printf("%x\n", f) // 0x1.edd3be22e5de1p+06

## 复数类型

比较局限和小众的类型，主要用于专业领域的计算，比如矢量计算等

Go 提供两种复数类型：

* complex64   实部与虚部都是 float32 类型
* complex128  实部与虚部都是 float64 类型

默认赋值类型为 complex128

### 复数字面值的表示

通过复数字面值直接初始化一个复数类型变量：

    var c = 5 + 6i
    var d = 0o123 + .12345E+5i // 83+12345i

通过 complex 函数创建一个 complex128 类型值：

    var c = complex(5, 6) // 5 + 6i
    var d = complex(0o123, .12345E+5) // 83+12345i

通过预定义函数 real 和 imag 获取一个复数的实部与虚部，返回一个浮点类型值：

    var c = complex(5, 6) // 5 + 6i
    r := real(c) // 5.000000
    i := imag(c) // 6.000000

## 创建自定义的数值类型

可以通过 type 关键字基于原生数值类型来声明一个新类型

通过 type 定义的新类型的底层类型虽然相同，但仍是两种完全不同的类型，无法直接相互赋值

新类型 MyInt 的数值性质与 int32 完全相同(同底层类型)：

    type MyInt int32
    
    var m int = 5
    var n int32 = 6
    var a MyInt = m // 错误：在赋值中不能将m（int类型）作为MyInt类型使用
    var a MyInt = n // 错误：在赋值中不能将n（int32类型）作为MyInt类型使用

借助**显式转换**，让赋值操作符左右两边的操作数保持类型一致，才能进行赋值：

    var m int = 5
    var n int32 = 6
    var a MyInt = MyInt(m) // ok
    var a MyInt = MyInt(n) // ok

## 通过类型别名（Type Alias）自定义数值类型

和 type 语法不同，通过类型别名语法定义的新类型与原类型完全等价，可以完全相互替代

不再需要显式转型，就可以相互赋值：

    type MyInt = int32

    var n int32 = 6
    var a MyInt = n // ok

## 字符串类型 string

Go 原生支持字符串，无论是字符串常量、字符串变量或是代码中出现的字符串字面值，它们的类型都被统一设置为 string

### string 类型的数据是不可变的，提高了字符串的并发安全性和存储利用率

    var s string = "hello"
    s[0] = 'k'   // 错误：字符串的内容是不可改变的
    s = "gopher" // ok 不能改变但是可以进行二次赋值

### 没有结尾’\0’，获取长度的时间复杂度是常数时间O(1)，消除了获取字符串长度的开销

Go 获取字符串长度是一个常数级时间复杂度，无论字符串中字符个数有多少，都可以快速得到字符串的长度值

### 原生支持“所见即所得”的原始字符串，大大降低构造多行字符串时的心智负担

反引号原生支持构造“所见即所得”的原始字符串，原始字符串中的任意转义字符都不会起到转义的作用

    var s string = `line1
        line2
        line3`
    fmt.Println(s)

### 对非 ASCII 字符提供原生支持，消除了源码在不同环境下显示乱码的可能

采用 Unicode 字符集，并且以 UTF-8 编码格式存储在内存当中

### Go 字符串的组成

**字节视角**

Go 语言中的字符串值也是一个可空的字节序列，字节序列中的字节个数称为该字符串的长度

一个个的字节只是孤立数据，不表意：

    var s = "中国人"
    fmt.Printf("the length of s = %d\n", len(s)) // 9<--这里是字节数

    for i := 0; i < len(s); i++ {
      fmt.Printf("0x%x ", s[i]) // 0xe4 0xb8 0xad 0xe5 0x9b 0xbd 0xe4 0xba 0xba
    }
    fmt.Printf("\n")

**字符串视角**

字符串是由一个可空的字符序列构成，表意：

    var s = "中国人"
    fmt.Println("the character count in s is", utf8.RuneCountInString(s)) // 3<--这里是字符数

    for _, c := range s {
      fmt.Printf("0x%x ", c) // 0x4e2d 0x56fd 0x4eba <--某种 Unicode 字符的表示，也就是Unicode 字符集表中的码点
    }
    fmt.Printf("\n")

### Unicode 码点

Unicode 字符集中的每个字符，都被分配了统一且唯一的字符编号

将 Unicode 字符集中的所有字符“排成一队”，字符在这个“队伍”中的位次，就是它在 Unicode 字符集中的码点

也就说，一个码点唯一对应一个字符

### rune 类型与字符字面值

Go 使用 rune 这个类型来表示一个 Unicode 码点

rune 本质上是 int32 类型的别名类型，与 int32 类型完全等价

    // $GOROOT/src/builtin.go
    type rune = int32

一个 rune 实例就是一个 Unicode 字符，一个 Go 字符串也可以被视为 rune 实例的集合

可以通过字符字面值来初始化一个 rune 变量

通过单引号括起的字符字面值：

    'a'  // ASCII字符
    '中' // Unicode字符集中的中文字符
    '\n' // 换行字符
    '\'' // 单引号字符

使用 Unicode 专用的转义字符\u 或\U 作为前缀，来表示一个 Unicode 字符：

    '\u4e2d'     // 字符：中  接两个十六进制数，两个不够则用大U表示
    '\U00004e2d' // 字符：中  可以接四个十六进制数
    '\u0027'     // 单引号字符

用整型值(rune 本质上就是一个整型数)直接作为字符字面值：

    '\x27'  // 使用十六进制表示的单引号字符
    '\047'  // 使用八进制表示的单引号字符

### 字符串字面值

用双引号包裹即可：

    "abc\n"
    "中国人"
    "\u4e2d\u56fd\u4eba" // 中国人
    "\U00004e2d\U000056fd\U00004eba" // 中国人
    "中\u56fd\u4eba" // 中国人，不同字符字面值形式混合在一起
    "\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba" // 十六进制表示的字符串字面值：中国人

### UTF-8 编码方案

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

### Go 字符串类型的内部表示⭐⭐⭐⭐⭐

从标准库 reflect 包中相关代码可以看到字符串的内部表示细节：

    // $GOROOT/src/reflect/value.go

    // StringHeader是一个string的运行时表示
    type StringHeader struct {
        Data uintptr <-- Data 指向一个底层数组
        Len  int <--字符串长度
    }

因为字符串类型中包含了字符串长度信息，所以获取长度的时间复杂度是常数时间，用 len 函数获取字符串长度时，len 函数只要简单地将这个信息提取出来就可以了

**string 类型其实是一个“描述符”，仅由一个指向底层存储的指针和字符串的长度字段组成，本身并不真正存储字符串数据**

直接将 string 类型通过函数 / 方法参数传入也不会带来太多的开销，因为传入的仅仅是一个“描述符”

打印字符串底层数组的内容：

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

### Go 字符串类型的常见操作

#### 下标操作

在字符串的实现中，真正存储数据的是底层的数组，所以字符串的下标操作本质上等价于对底层数组的下标操作：

    var s = "中国人"
    fmt.Printf("0x%x\n", s[0]) // 0xe4：字符“中” utf-8编码的第一个字节
    // 注意通过下标操作，我们获取的是字符串中特定下标上的字节，而不是字符

#### 字符迭代

通过常规 for 迭代，是一种字节视角的迭代，等价于对字符串底层数组的迭代：

    var s = "中国人"

    for i := 0; i < len(s); i++ {
      // 每轮迭代得到的的结果都是组成字符串内容的一个字节，以及该字节所在的下标值
      fmt.Printf("index: %d, value: 0x%x\n", i, s[i])
    }

通过 for range 迭代，每轮迭代得到的是字符串中 Unicode 字符的码点值，以及该字符在字符串中的偏移值：

    var s = "中国人"

    for i, v := range s {
        fmt.Printf("index: %d, value: 0x%x\n", i, v)
    }

**可以看到通过这两种形式的迭代对字符串进行操作得到的结果是不同的**

#### 获取字符串长度

len() 获取的是字符串的字节长度

用UTF-8 包中的 RuneCountInString 函数可以获取字符串中字符个数

#### 字符串连接

虽然字符串内容是不可变的，但是可以基于已有字符串创建新字符串

Go 原生支持通过 +/+= 操作符进行字符串连接：

    s := "Rob Pike, "
    s = s + "Robert Griesemer, "
    s += " Ken Thompson"

    fmt.Println(s) // Rob Pike, Robert Griesemer, Ken Thompson

虽然通过 +/+= 进行字符串连接的开发体验是最好的，但连接性能就未必是最快的

其他字符串连接操作函数：strings.Builder、strings.Join、fmt.Sprintf 

如果能知道拼接字符串的个数，那么使用 bytes.Buffer 和 strings.Builder 的 Grows 申请空间后，性能是最好的；如果不能确定长度，那么 bytes.Buffer 和 strings.Builder 也比 “+” 和 fmt.Sprintf 性能好很多

其中 bytes.Buffer 与 strings.Builder 相比，strings.Builder 更合适，因为 bytes.Buffer 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 strings.Builder 直接将底层的 []byte 转换成了字符串类型然后返回

bytes.Buffer 的注释中还特意提到了：

    To build strings more efficiently, see the strings.Builder type.

综合易用性和性能，一般推荐使用 strings.Builder 来拼接字符串：

    A Builder is used to efficiently build a string using Write methods. It minimizes memory copying.

#### 字符串比较

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

鉴于 Go string 类型是不可变的，所以说如果两个字符串的长度不相同，就直接可以断定两个字符串是不同的

如果两个字符串长度相同，就要进一步判断，数据指针是否指向同一块底层存储数据

如果还相同，那么我们可以说两个字符串是等价的，如果不同，那就还需要进一步去比对实际的数据内容

**简而言之就是先比长度，然后指针，然后比较实际数据内容**

#### 字符串转换

Go 支持字符串与字节切片、字符串与 rune 切片的双向转换，使用显式类型转换就可以无需调用任何函数：

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

string 转切片，或者切片转 string，都有一定的性能开销，因为string 是不可变的，运行时要为转换后的类型分配新内存

# 常量

Go 语言常量的几个特点：

* 支持无类型常量
* 支持隐式自动转型
* 可用于实现枚举

Go 语言的常量是一种在源码编译期间被创建的语法元素，这个元素的值可以像变量那样被初始化，但初始化的表达式必须是在编译期间可以求出值来的，一旦声明并被初始化后，它的值在整个程序的生命周期内便保持不变，被创建并初始化后的常量还可以作为其他常量的初始表达式的一部分

使用const 关键字来声明常量：

    const Pi float64 = 3.14159265358979323846 // 单行常量声明

    // 以const代码块形式声明常量
    const (
        size int64 = 4096
        i, j, s = 13, 14, "bar" // 单行声明多个常量
    )

Go 常量的类型只局限于 Go 基本数据类型，包括数值类型、字符串类型，以及布尔类型

## 无类型常量

有类型常量中，即便两个类型拥有着相同的底层类型，只要它们不是用**同名类型**声明的，就不能互相赋值或者进行运算，需要通过显式转型后再进行操作

    type myInt int
    const n myInt = 13
    const m int = n + 5 // 编译器报错：cannot use n + 5 (type myInt) as type int in const initializer
                        // const m int = n + myInt(5) 常量中不能进行显式转换，这样做是错误的

    func main() {
        var a int = 5
        fmt.Println(a + n) // 编译器报错：invalid operation: a + n (mismatched types int and myInt)
        // fmt.Println(myInt(a) + n) 显示转换后则可以相加
    }

如果不给常量定义类型，就会自动隐式的进行转换，并根据赋予的初值形式来决定默认类型，称之为无类型常量（Untyped Constant）：

    type myInt int
    const n = 13

    func main() {
        var a myInt = 5
        fmt.Println(a + n)  // 输出：18 自动将 a+n 表达式中的常量 n 转型为 myInt 类型，再与变量 a 相加
    }

无类型常量不是真的没有类型，它也有自己的默认类型，默认类型是根据初值形式来决定的

## 隐式转型

对于无类型常量参与的表达式求值，Go 编译器会根据上下文中的类型信息，把无类型常量自动转换为相应的类型后，再参与求值计算，这一转型动作是隐式进行的

对常量进行转型并不会引发类型安全问题，Go 编译器会保证这一转型的安全性

如果无法将常量转换为目标类型，Go 编译器会报错：

    const m = 1333333333 // 已经超出了 int8 类型可以表示的范围，整形溢出

    var k int8 = 1
    j := k + m // 编译器报错：constant 1333333333 overflows int8

无类型常量与常量隐式转型的互相结合，在处理表达式混合数据类型运算的时候具有比较大的灵活性，代码编写也得到了一定程度的简化

也就不需要在求值表达式中做任何显式转型了

在 Go 中，使用无类型常量是一种惯用法

## 实现枚举

Go 语言并没有原生提供枚举类型，而使用 const 代码块定义的常量集合，来实现枚举

枚举类型本质上就是一个由**有限数量常量**所构成的集合，所以这样做并没有什么问题

Go 枚举特性：

* 默认不是和C语言一样每行 +1 的，而是**自动重复上一行**
* 引入 const 块中的行偏移量指示器 iota

### 自动重复上一行

    const (
        Apple, Banana = 11, 22
        Strawberry, Grape  = 11, 22 // 使用上一行的初始化表达式
        Pear, Watermelon  = 11, 22 // 使用上一行的初始化表达式
    )

### 偏移量指示器 iota

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

虽然 iota 带来了灵活性与便利，但也有一些场合不适于使用 iota

比如标准库 syscall ，因为系统调用的编号以及错误码值几乎形成了标准，直接用原值更好，也方便查找，如果使用iota会看得头晕，心智负担太大

# 指针类型

指针类型是依托某一个**基类型**而存在的，比如：一个整型为 int，那么它对应的整型指针就是 *int，没有 int 类型，就不会有 *int 类型

**如果拥有一个类型 T，那么以 T 作为基类型的指针类型为 *T**

    var p *T

**unsafe.Pointer** 类似于 C 语言中的 void*，不需要基类型，用于表示一个通用指针类型，**也就是任何指针类型都可以显式转换为一个 unsafe.Pointer，而 unsafe.Pointer 也可以显式转换为任意指针类型**：

    var p *T
    var p1 = unsafe.Pointer(p) // 任意指针类型显式转换为unsafe.Pointer
    p = (*T)(p1)               // unsafe.Pointer也可以显式转换为任意指针类型

如果指针类型变量没有被显式赋予初值，那么它的值为 **nil**：

    var p *T
    println(p == nil) // true

给一个指针类型变量赋值：

    var a int = 13
    var p *int = &a  // 给整型指针变量p赋初值

变量 a 前面的&符号称为**取地址符号**，只能使用基类型变量的地址给对应的指针类型变量赋值，如果类型不匹配，Go 编译器是会报错：

    var b byte = 10
    var p *int = &b // Go编译器报错：cannot use &b (value of type *byte) as type *int in variable declaration 

**对于非指针类型变量，Go 在对应的内存单元中放置的就是该变量的值**

**指针变量分配的内存单元中存储的是被引用变量对应的内存单元的地址**

指针类型变量的大小与其基类型大小无关，而是和系统地址的表示长度有关，可以通过 unsafe.Sizeof() 函数获取：

    var p1 *int
    println(unsafe.Sizeof(p1)) // 8

unsafe.Sizeof() 函数原型如下：

    func Sizeof(x ArbitraryType) uintptr

这个函数的返回值类型是 uintptr，**uintptr 是一个整数类型，它的大小足以容纳任何指针的比特模式（bit pattern），在 Go 语言中 uintptr 类型的大小就代表了指针类型的大小**

通过指针读取或修改其指向的内存单元所代表的基类型变量：

    var a int = 17
    var p *int = &a
    println(*p) // 17 
    (*p) += 3
    println(a)  // 20

通过在指针类型变量的前面加上一个**星号**，读取或修改其指向的内存地址上的变量值，这种操作被称为指针的**解引用（dereference）**

通过**解引用**输出或修改的，并不是指针变量本身的值，而是指针指向的内存单元的值

可以使用 Printf 通过 %p 输出指针自身的值：

    fmt.Printf("%p\n", p) // 0xc0000160d8

指针变量重新赋值，可以变换其指向的内存单元：

    var a int = 5
    var b int = 6

    var p *int = &a  // 指向变量a所在内存单元
    println(*p)      // 输出变量a的值
    p = &b           // 指向变量b所在内存单元
    println(*p)      // 输出变量b的值

多个指针变量指向同一个变量的内存单元，通过其中一个指针变量对内存单元的修改，可以通过另外一个指针变量的解引用反映出来：

    var a int = 5
    var p1 *int = &a // p1指向变量a所在内存单元
    var p2 *int = &a // p2指向变量b所在内存单元
    (*p1) += 5       // 通过p1修改变量a的值
    println(*p2)     // 10 对变量a的修改可以通过另外一个指针变量p2的解引用反映出来

## 二级指针

    package main

    func main() {
        var a int = 5
        var p1 *int = &a
        println(*p1) // 5
        var b int = 55
        var p2 *int = &b
        println(*p2) // 55

        var pp **int = &p1 // 二级指针
        println(**pp) // 5
        pp = &p2      
        println(**pp) // 55
    }

二级指针也就是指向指针的指针

对一级指针解引用，得到的其实是指针指向的变量，而对二级指针 pp 解引用一次，我们得到将是 pp 指向的指针变量

    println((*pp) == p1) // true

解引用两次，其实就相当于对一级指针解引用一次，得到的是 pp 指向的指针变量所指向的整型变量：

    println((**pp) == (*p1)) // true
    println((**pp) == a)     // true

一级指针常被用来改变普通变量的值，**二级指针可以用来改变指针变量的值，也就是指针变量的指向**

## Go 中的指针用途与使用限制

### 限制一：限制了显式指针类型转换

    var a int = 0x12345678
    var pa *int = &a
    var pb *byte = (*byte)(pa) // 编译器报错：cannot convert pa (variable of type *int) to type *byte
    fmt.Printf("%x\n", *pb)

通过 unsafe 的方式转换：

    var a int = 0x12345678                                                            
    var pa *int = &a                                                                  
    var pb *byte = (*byte)(unsafe.Pointer(pa)) // ok
    fmt.Printf("%x\n", *pb) // 78   

### 限制二：不支持指针运算

    var arr = [5]int{1, 2, 3, 4, 5}
    var p *int = &arr[0]
    println(*p)
    p = p + 1  // 编译器报错：cannot convert 1 (untyped int constant) to *int
    println(*p)

通过 unsafe 的方式计算：

    var arr = [5]int{11, 12, 13, 14, 15}
    var p *int = &arr[0]
    var i uintptr
    for i = 0; i < uintptr(len(arr)); i++ {
        p1 := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + i*unsafe.Sizeof(*p)))
        println(*p1)
    }           