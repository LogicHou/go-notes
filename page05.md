# 05 复合数据类型

## 定长数组

* 长度固定
* 类型同构

由**元素的类型**和**数组长度**（元素的个数）构成了数组类型变量的声明：

    var arr [N]T
    # 声明了一个数组变量 arr，它的类型为 [N]T
    # N代表长度，必须在声明时提供
    # T数组类型，可以为任意的 Go 原生类型或自定义类型(type)

N 和 T 共同构成了数组的类型

如果两个数组类型的元素类型 T 与数组长度 N 都是一样的，那么这两个数组类型是等价的

如果有一个属性不同，就是两个不同的数组类型：

    func foo(arr [5]int) {}
    func main() {
        var arr1 [5]int
        var arr2 [6]int
        var arr3 [5]string

        foo(arr1) // ok 类型相同所以类型转换成功
        foo(arr2) // 错误：[6]int与函数foo参数的类型[5]int不是同一数组类型
        foo(arr3) // 错误：[5]string与函数foo参数的类型[5]int不是同一数组类型
    }  

数组是逻辑连续的序列，在实际内存分配时占据着一整块内存，Go 编译器会为数组分配一整块、可以容纳它所有元素的连续内存

可以通过 len 函数获取一个数组的长度，通过 unsafe 包提供的 Sizeof 函数，获取一个数组变量内存分配的总大小：

    var arr = [6]int{1, 2, 3, 4, 5, 6}
    fmt.Println("数组长度：", len(arr))           // 6
    fmt.Println("数组大小：", unsafe.Sizeof(arr)) // 48 数组大小就是所有元素占用内存大小之和, 6x8=48 个字节

声明一个数组类型变量的同时，可以显式地对它进行初始化，如果不进行显式初始化，数组中的元素值就是它类型的零值初始化：
    
    var arr1 [6]int // [0 0 0 0 0 0] 没有显式初始化，默认为元素值类型的零值
    
    var arr2 = [6]int {
        11, 12, 13, 14, 15, 16,
    } // [11 12 13 14 15 16] 进行了显示初始化

    var arr3 = [...]int { // “...” 表示由编译器自动计算出数组长度
        21, 22, 23,
    } // [21 22 23]
    fmt.Printf("%T\n", arr3) // [3]int

    var arr4 = [...]int{
        99: 39, // 将第100个元素(下标值为99)的值赋值为39，其余元素值均为0
    }
    fmt.Printf("%T\n", arr4) // [100]int

数组的访问，**下标值是从 0 开始的**：

    var arr = [6]int{11, 12, 13, 14, 15, 16}
    fmt.Println(arr[0], arr[5]) // 11 16
    fmt.Println(arr[-1])        // 错误：下标值不能为负数
    fmt.Println(arr[8])         // 错误：小标值超出了arr的长度范围

多维数组：

    var mArr [2][3][4]int

数组的不足之处

* 固定的元素个数，一个数组变量表示的是整个数组是一个整体
* 传值机制，无论是参与迭代，还是作为实参传给一个函数 / 方法，都是纯粹的值拷贝
* 会带来较大的内存拷贝开销，可以通过传引用解决，但是这种方式数组依旧无法灵活的伸缩扩展
* 推荐使用更为灵活、更为地道的方式 ，切片 slice

## 变长切片 slice

切片的声明与数组相比仅仅是少了一个“长度”属性：

    var nums = []int{1, 2, 3, 4, 5, 6}
    var sl1 []int     // 还没初始化，是nil值，底层没有分配内存空间，也是 go官方推荐使用的方式
    var sl2 = []int{} // 初始化了，不是nil值，底层分配了内存空间，有地址

通过 len 函数获得切片类型变量的长度：

    fmt.Println(len(nums)) // 6

通过内置函数 append 动态地向切片中添加元素：

    nums = append(nums, 7) // 切片变为[1 2 3 4 5 6 7]
    fmt.Println(len(nums)) // 7

切片中的 “...” （打散）：

    arr1 := []int{1, 2, 3}
    arr2 := []int{4, 5, 6}
    arr1 = append(arr1, arr2...)
    fmt.Printf("append: %v\n\n", arr1) // 输出 [1, 2, 3, 4, 5, 6]

### 切片类型的内部实现

    type slice struct {
        array unsafe.Pointer // array: 是指向底层数组的指针
        len   int            //   len: 是切片的长度，即切片中当前元素的个数
        cap   int            //   cap: 是底层数组的长度，也是切片的最大容量，cap 值永远大于等于 len 值
    }

### 创建切片的几种方法

**通过 make 函数创建切片，并指定底层数组的长度**：

    sl := make([]byte, 6, 10) // 其中10为cap值，即底层数组长度，6为切片的初始长度

    sl := make([]byte, 6) // cap = len = 6 没有指定 cap 参数，那么底层数组长度 cap 就等于 len

**用 array[low : high : max] 语法基于一个已存在的数组创建切片，被称为数组的切片化**：

    arr := [10]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    sl := arr[3:7:9] // [4，5，6，7]
        
    sl[0] += 10
    fmt.Println("arr[3] =", arr[3]) // 14 改变了底层数组的值

* 起始元素从 low 所标识的下标值开始
* 切片的长度(len)是 high - low
* 容量(cap)是 max - low
* 切片 sl 的底层数组就是数组 arr，对切片 sl 中元素的修改将直接影响数组 arr 变量
* 切片之于数组就像是文件描述符之于文件，传递的并不是数组本身，而是一个大小固定的数组的“描述符”，开销小
* 在进行数组切片化的时候，通常省略 max，而 max 的默认值为数组的长度
* 针对一个已存在的数组，可以建立多个操作数组的切片，这些切片共享同一底层数组，对底层数组的操作也同样会反映到其他切片中

**基于切片创建切片**：

原理与上面一样，但是要记得默认的 cap 是 max - low

    arr1 := [...]int{1, 2, 3, 4, 5, 6}
    arr2 := arr1[1:3]
    fmt.Println(arr2, len(arr2), cap(arr2)) // [2 3] 2 5


### 切片的动态扩容

切片的“动态扩容”需要 Go 运行时提供支持，这种不定长的特性是与数组最大的不同

通过 append 操作向切片追加数据的时候，如果切片的 len 值和 cap 值是相等的，则表示底层数组空间已满不能再存储追加的值，Go 运行时就会对这个切片做扩容操作，来保证切片始终能存储下追加的新值：

    var s []int  // 初值为零值（nil），还没有“绑定”底层数组
    s = append(s, 11) 
    fmt.Println(len(s), cap(s)) //1 1
    s = append(s, 12) 
    fmt.Println(len(s), cap(s)) //2 2
    s = append(s, 13) 
    fmt.Println(len(s), cap(s)) //3 4
    s = append(s, 14) 
    fmt.Println(len(s), cap(s)) //4 4
    s = append(s, 15) 
    fmt.Println(len(s), cap(s)) //5 8

append 操作的自动扩容行为，一旦追加的数据操作触碰到切片的容量上限（实质上也是数组容量的上界)，切片就会和原数组**解除“绑定”**，后续对切片的任何修改都不会反映到原数组中了：

    u := [...]int{11, 12, 13, 14, 15}
    fmt.Println("array:", u) // [11, 12, 13, 14, 15]
    s := u[1:3]
    fmt.Printf("slice(len=%d, cap=%d): %v\n", len(s), cap(s), s) // [12, 13]
    s = append(s, 24)
    fmt.Println("after append 24, array:", u)
    fmt.Printf("after append 24, slice(len=%d, cap=%d): %v u[0]ptr:%p, s[0]ptr:%p\n", len(s), cap(s), s, &u[1], &s[0])
    s = append(s, 25)
    fmt.Println("after append 25, array:", u)
    fmt.Printf("after append 25, slice(len=%d, cap=%d): %v u[0]ptr:%p, s[0]ptr:%p\n", len(s), cap(s), s, &u[1], &s[0])
    s = append(s, 26)
    fmt.Println("after append 26, array:", u)
    fmt.Printf("after append 26, slice(len=%d, cap=%d): %v u[0]ptr:%p, s[0]ptr:%p\n", len(s), cap(s), s, &u[1], &s[0])

    s[0] = 22
    fmt.Println("after reassign 1st elem of slice, array:", u)
    fmt.Printf("after reassign 1st elem of slice, slice(len=%d, cap=%d): %v\n", len(s), cap(s), s)


得到结果：

    array: [11 12 13 14 15]
    slice(len=2, cap=4): [12 13]
    after append 24, array: [11 12 13 24 15]
    after append 24, slice(len=3, cap=4): [12 13 24] u[1]ptr:0xc00001a2d8, s[0]ptr:0xc00001a2d8
    after append 25, array: [11 12 13 24 25]
    after append 25, slice(len=4, cap=4): [12 13 24 25] u[1]ptr:0xc00001a2d8, s[0]ptr:0xc00001a2d8 <-- 注意和下面相比
    after append 26, array: [11 12 13 24 25]
    after append 26, slice(len=5, cap=8): [12 13 24 25 26] u[1]ptr:0xc00001a2d8, s[0]ptr:0xc00001c080 <-- 注意这里的内存地址和上面相比已经变了
    after reassign 1st elem of slice, array: [11 12 13 24 25]
    after reassign 1st elem of slice, slice(len=5, cap=8): [22 13 24 25 26]

## map 类型

表示一组无序的键值对，集合中每个 key 都是唯一的，由 key 类型与 value 类型组成：

    map[key_type]value_type

key 与 value 的类型可相同或不同：

    map[string]string // key与value元素的类型相同
    map[int]string    // key与value元素的类型不同

map 类型对 value 的类型没有限制

但是 key 必须保证唯一性，因此**key 的类型必须支持“==”和“!=”两种比较操作符**

如果两个 map 类型的 key 和 value 元素类型都相同，可以说它们是同一个 map 类型

在 Go 语言中，函数类型、map 类型自身，以及切片只支持与 nil 的比较，而不支持同类型两个变量的比较：

    s1 := make([]int, 1)
    s2 := make([]int, 2)
    f1 := func() {}
    f2 := func() {}
    m1 := make(map[int]string)
    m2 := make(map[int]string)
    println(s1 == s2) // 错误：invalid operation: s1 == s2 (slice can only be compared to nil)
    println(f1 == f2) // 错误：invalid operation: f1 == f2 (func can only be compared to nil)
    println(m1 == m2) // 错误：invalid operation: m1 == m2 (map can only be compared to nil)

所以**函数类型、map 类型自身，以及切片类型是不能作为 map 的 key 类型的**

### 声明

如果没有显式地赋予 map 变量初值，map 类型变量的默认值为 nil

    var m map[string]int // 一个map[string]int类型的变量

和切片变量不同的是，初值为零值 nil 的切片类型变量，可以借助内置的 append 的函数进行操作，在 Go 中称之为**“零值可用”**
  
关于零值可用：

* 可以提升开发者的使用体验
* 不用再担心变量的初始状态是否有效

map 类型因为内部实现的复杂性，无法“零值可用”

对处于零值状态的 map 变量直接进行操作，就会导致运行时异常（panic），导致程序进程异常退出：

    var m map[string]int // m = nil
    m["key"] = 1         // 发生运行时异常：panic: assignment to entry in nil map

所以 map 类型变量必须显式初始化后才能使用

### 初始化

使用复合字面值初始化 map 类型变量：

    m := map[int]string{} // 此时变量 m 中没有任何键值对，但也不等同于初值为 nil 的 map 变量
                          // 此时进行键值对的插入操作，不会引发运行时异常

复杂一点的：

    m1 := map[int][]string{
        1: []string{"val1_1", "val1_2"},
        3: []string{"val3_1", "val3_2", "val3_3"},
        7: []string{"val7_1"},
    }

    type Position struct { // 这里是一个结构体
        x float64 
        y float64
    }

    m2 := map[Position]string{
        Position{29.935523, 52.568915}: "school",
        Position{25.352594, 113.304361}: "shopping-mall",
        Position{73.224455, 111.804306}: "hospital",
    }

允许省略字面值中的元素类型，编译器会自行推导：

    m2 := map[Position]string{
        {29.935523, 52.568915}: "school",
        {25.352594, 113.304361}: "shopping-mall",
        {73.224455, 111.804306}: "hospital",
    }

使用 make 进行显式初始化，且初始化后不是 nil 值：

    m1 := make(map[int]string) // 未指定初始容量
    m2 := make(map[int]string, 8) // 指定初始容量为8

map 类似的容量不受限于它的初始容量值，当元素数量超过初始容量后，也会自动进行扩容

### 遍历 slice 元素

使用 for range 语句进行遍历：

    sl := []int{1, 2, 3, 4, 5, 6}
    for k, v := range sl {
      println(k, v)
    }

    输出
    0 1
    1 2
    2 3
    3 4
    4 5
    5 6

### map 的基本操作

#### 插入新键值对

首先插入的对象必须是非 nil 的 map 类型变量，才可以在其中插入符合 map 类型定义的任意新键值对

插入操作很简单，只需把 value 直接赋值给对应的 key 即可：

    m := make(map[int]string)
    m[1] = "value1"
    m[2] = "value2"
    m[3] = "value3"

除非系统内存耗尽，否则插入总是成功的，插入的时候如果 key 已经存在，会用新值覆盖旧值：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    m["key1"] = 11 // 11会覆盖掉"key1"对应的旧值1
    m["key3"] = 3  // 此时m为map[key1:11 key2:2 key3:3]

#### 获取键值对数量

通过 len 函数：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    fmt.Println(len(m)) // 2
    m["key3"] = 3  
    fmt.Println(len(m)) // 3

和切片类型不同，不能对 map 类型变量调用 cap 函数来获取当前容量

#### 查找和数据读取

尝试获取一个键对应的值的时候，如果这个 key 在 map 中并不存在，会得到这个 value 元素类型的**零值**：

    m := make(map[string]int)
    v := m["key1"] 

如果 key1 不存在，就会被赋予 int 的零值，也就是 0

所以无法通过 v 值判断出，究竟是因为 key1 不存在返回的零值，还是因为 key1 本身对应的 value 就是 0

使用 “comma ok” 惯用法对 map 进行键查找和键值读取操作：

    m := make(map[string]int)
    v, ok := m["key1"]
    if !ok {
        // "key1"不在map中
    }
    // "key1"在map中，v将被赋予"key1"键对应的value

    
    m := make(map[string]int)
    _, ok := m["key1"] // 用 _ 忽略可能返回的 value
    ... ...

#### 删除数据

使用**内置函数 delete** 从 map 中删除数据：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    fmt.Println(m) // map[key1:1 key2:2]
    delete(m, "key2") // 删除"key2"
    fmt.Println(m) // map[key1:1]

delete 函数是从 map 中删除键的**唯一方法**

即便传给 delete 的键在 map 中并不存在，delete 函数的执行也不会失败，更不会抛出运行时的异常

#### 遍历 map 中的键值数据

唯一遍历 map 的方法 for range 语句：

    m := map[int]int{
        1: 11,
        2: 12,
        3: 13,
    }

    for k, v := range m { // 每次迭代都会返回一个键值对
        fmt.Printf("[%d, %d] ", k, v) // 键存在于变量 k 中，对应的值存储在变量 v 中
    }

只关心每次迭代的键：

    for k, _ := range m { 
      // 使用k
    }

    for k := range m {
      // 使用k
    }

只关心每次迭代返回的键所对应的 value ：

    for _, v := range m {
      // 使用v
    }

关于 map 一个很重要的问题就是，**对同一 map 做多次遍历的时候，每次遍历元素的次序都不相同**

这是一个坑，所以 Go **程序逻辑千万不要依赖遍历 map 所得到的的元素次序**

### map 变量的传递开销

和切片类型一样，map 也是引用类型，实质上传递的也只是一个“描述符”，传递的开销是固定的，而且也很小

而且以描述符传递的类型(map slice 引用 )，在函数或者方法内部中的修改对外部也是可见的

    func foo(m map[string]int) {
        m["key1"] = 11
        m["key2"] = 12
    }

    func main() {
        m := map[string]int{
            "key1": 1,
            "key2": 2,
        }

        fmt.Println(m) // map[key1:1 key2:2]  
        foo(m)
        fmt.Println(m) // map[key1:11 key2:12] 
    }

### map 的内部实现

Go 运行时使用一张哈希表来实现抽象的 map 类型，运行时实现了 map 类型的所有操作，包括查找、插入、删除等

Go 编译器会将语法层面的 map 操作，**重写成运行时对应的函数调用**：

    // 创建map类型变量实例
    m := make(map[keyType]valType, capacityhint) → m := runtime.makemap(maptype, capacityhint, m)

    // 插入新键值对或给键重新赋值
    m["key"] = "value" → v := runtime.mapassign(maptype, m, "key") v是用于后续存储value的空间的地址

    // 获取某键的值
    v := m["key"]      → v := runtime.mapaccess1(maptype, m, "key")
    v, ok := m["key"]  → v, ok := runtime.mapaccess2(maptype, m, "key")

    // 删除某键
    delete(m, "key")   → runtime.mapdelete(maptype, m, “key”)

### map 与并发

map 实例不是并发写安全的，也不支持并发读写，如果对 map 实例进行并发读写，程序运行时就会抛出异常

如果仅仅是进行并发读，map 是没有问题的

并发写安全的 sync.Map 类型(Go version >= 1.9)，可以用来在并发读写的场景下替换掉 map

注意，因为 map 可以自动扩容，map 中数据元素的 value 位置可能在这一过程中发生变化

所以 **Go 不允许获取 map 中 value 的地址，这个约束在编译期间就生效**，如果对 value 取地址就会报编译错误：

    p := &m[key]  // cannot take the address of m[key]
    fmt.Println(p)

## 结构体

### 定义一个新类型

#### 使用关键字type进行类型定义（Type Definition）

    type T S // 定义一个新类型T

S 可以是任何一个已定义的类型，包括 Go 原生类型，或者是其他已定义的自定义类型：

    type T1 int // 底层类型为int
    type T2 T1  // 基于T1，T1的底层类型为int，所以T2的底层类型同为int

如果一个新类型是基于某个 Go 原生类型定义的，那么我们就叫 Go 原生类型为新类型的**底层类型**，被用来判断两个类型本质上是否相同

T1 和 T2 是不同类型，但底层类型都是类型 int，所以本质上是相同的

而本质上相同的两个类型，它们的变量可以通过显式转型进行相互赋值

相反，如果本质上是不同的两个类型，它们的变量间连显式转型都不可能，更不能相互赋值

基于类型字面值来定义新类型，多用于自定义一个新的复合类型：

    type M map[int]string
    type S []string

#### 使用类型别名（Type Alias）

通常用在项目的渐进式重构，还有对已有包的二次封装方面，形式：

    type T = S // type alias

和类型定义相比，类型别名的形式只是多了一个等号，用类型别名定义的新类型 T 与原类型 S 完全等价，也就是并没有定义出新类型，T 与 S 实际上就是**同一种类型**

### 定义一个结构体类型

典型的结构体类型的定义形式：

    type T struct {
        Field1 T1
        Field2 T2
        ... ...
        FieldN Tn
    }


    type Book struct {
        Title string              // 书名
        Pages int                 // 书的页数
        Indexes map[string]int    // 书的索引
    }

得到一个名为 T 的结构体类型，定义中 struct 关键字后面的大括号包裹的内容就是一个**类型字面值**

由若干个字段（field）聚合而成，每个字段有自己的名字与类型，并且在一个结构体中，每个字段的名字应该都是唯一的

首字母大写的是导出标识符，可以被其他包所使用

可以用空标识符“_”作为结构体类型定义中的字段名称，不能被外部包引用，也无法被结构体所在的包使用

### 特殊情况

#### 定义一个空结构体

    type Empty struct{} // Empty是一个不包含任何字段的空结构体类型
    
    var s Empty
    println(unsafe.Sizeof(s)) // 0 空结构体类型变量的大小为 0

空结构体类型变量的内存占用为 0，基于这个内存零开销的特性，可以将其作为一种“事件”信息进行 Goroutine 之间的通信：

    var c = make(chan Empty) // 声明一个元素类型为Empty的channel
    c<-Empty{}               // 向channel写入一个“事件”

这种以空结构体为元素类建立的 channel，是目前能实现的、内存占用最小的 Goroutine 间通信方式

#### 使用其他结构体作为自定义结构体中字段的类型

    type Person struct {
        Name string
        Phone string
        Addr string
    }

    type Book struct {
        Title string
        Author Person
        ... ...
    }

访问 Book 结构体字段 Author 中的 Phone 字段：

    var book Book 
    println(book.Author.Phone)

嵌入字段（Embedded Field）或者叫匿名字段，是一种更为简便的定义方法，可以无需提供字段的名字，只使用其类型就可以：

    type Book struct {
        Title string
        Person
        ... ...
    }

访问 Person 中的 Phone 字段：

    var book Book 
    println(book.Person.Phone) // 将类型名当作嵌入字段的名字
    println(book.Phone)        // 支持直接访问嵌入字段所属类型中字段

Go 语言不支持在结构体类型定义中，递归地放入其自身类型字段的定义方式：

    type T struct {
        t T  
        ... ...
    }
    
    type T1 struct {
      t2 T2
    }

    type T2 struct {
      t1 T1
    }
    // 编译器就会给出“invalid recursive type T”的错误信息

但可以拥有自身类型的指针类型、以自身类型为元素类型的切片类型，以及以自身类型作为 value 类型的 map 类型的字段：

    type T struct {
        t  *T           // ok
        st []T          // ok slice 是描述符类型，不用加 * 号
        m  map[string]T // ok map   也是描述符类型，不用加 * 号
    }

### 结构体变量的声明与初始化

    type Book struct {
        ...
    }

    var book Book
    var book = Book{}
    book := Book{}

结构体类型通常是对真实世界复杂事物的抽象，这和简单的数值、字符串、数组 / 切片等类型有所不同

**结构体类型的变量通常都要被赋予适当的初始值后，才会有合理的意义**

#### 零值初始化

    var book Book // book为零值结构体变量

这样定义的一本书既没有书名，也没有作者、页数、索引等信息，对这本书的抽象就失去了实际价值

对于像 Book 这样的结构体类型，使用零值初始化并不是正确的选择

但是采用零值初始化的零值结构体变量也有其存在的意义

如果一种类型采用零值初始化得到的零值变量，是有意义的，而且是直接可用的，我称这种类型为“零值可用”类型

定义零值可用类型是简化代码、改善开发者使用体验的一种重要的手段

比如标准库 sync 包的 Mutex 类型：

    var mu sync.Mutex
    mu.Lock()
    mu.Unlock()

比如标准库 bytes.Buffer 结构体类型：

    var b bytes.Buffer
    b.Write([]byte("Hello, Go"))
    fmt.Println(b.String()) // 输出：Hello, Go

#### 使用复合字面值

在日常开发中，对结构体类型变量进行显式初始化的最常用方法就是使用复合字面值

最简单的就是按顺序依次给每个结构体字段进行赋值，比如下面的代码：

    type Book struct {
        Title string              // 书名
        Pages int                 // 书的页数
        Indexes map[string]int    // 书的索引
    }

    var book = Book{"The Go Programming Language", 700, make(map[string]int)}

但是这存在还多问题：

* 当结构体类型定义中的字段顺序发生变化，或者字段出现增删操作时，我们就需要手动调整该结构体类型变量的显式初始化代码，让赋值顺序与调整后的字段顺序一致
* 当一个结构体的字段较多时，这种逐一字段赋值的方式实施起来就会比较困难，而且容易出错，开发人员需要来回对照结构体类型中字段的类型与顺序，谨慎编写字面值表达式
* 一旦结构体中包含非导出字段，那么这种逐一字段赋值的方式就不再被支持了

强烈不推荐使用这样的方式：

    type T struct {
        F1 int
        F2 string
        f3 int
        F4 int
        F5 int
    }

    var t = T{11, "hello", 13} // 错误：implicit assignment of unexported field 'f3' in T literal
    或
    var t = T{11, "hello", 13, 14, 15} // 错误：implicit assignment of unexported field 'f3' in T literal

Go 推荐使用“field:value”形式的复合字面值：

    var t = T{
        F2: "hello",
        F1: 11,
        F4: 14,
    }

采用类型零值初始化：

    t := T{}

使用预定义函数 new 创建结构体变量实例：

    tp := new(T)

#### 使用特定的构造函数

解决一个结构体类型中包含未导出字段，并且这个字段的零值还不可用，或者一个结构体类型中的某些字段，需要一个复杂的初始化逻辑

比如标准库中的 time.Timer 这个结构体就是一个典型的例子：

    // $GOROOT/src/time/sleep.go
    type runtimeTimer struct {
        pp       uintptr
        when     int64                       // 零值不可用
        period   int64
        f        func(interface{}, uintptr)  // 零值不可用
        arg      interface{}                 // 零值不可用
        seq      uintptr
        nextwhen int64
        status   uint32
    }

    type Timer struct {
        C <-chan Time
        r runtimeTimer
    }

这个 runtimeTimer 结构体中的一些字段不是零值可用的，所以标准库提供了一个 Timer 结构体专用的构造函数 NewTimer：

    // $GOROOT/src/time/sleep.go
    func NewTimer(d Duration) *Timer {
        c := make(chan Time, 1)
        t := &Timer{
            C: c,
            r: runtimeTimer{
                when: when(d),
                f:    sendTime,
                arg:  c,
            },
        }
        startTimer(&t.r)
        return t
    }

专用构造函数大多都符合这种模式：

    func NewT(field1, field2, ...) *T {
        ... ...
    }

参数列表中的参数通常与 T 定义中的导出字段相对应，返回值则是一个 T 指针类型的变量

T 的非导出字段在 NewT 内部进行初始化，一些需要复杂初始化逻辑的字段也会在 NewT 内部完成初始化

这样，只要调用 NewT 函数就可以得到一个可用的 T 指针类型变量了

### 结构体类型的内存布局

是既数组类型之后，第二个将元素（结构体字段）一个接着一个以“平铺”形式，存放在一个连续内存块中

在真实情况下，结构体字段实际上可能并不是紧密相连的，中间可能存在“缝隙”

这些“缝隙”同样是结构体变量占用的内存空间的一部分，是 Go 编译器插入的“填充物（Padding）”

插入“填充物”的目的是为了满足**内存对齐**的要求，所谓内存对齐，指的就是各种内存对象的内存地址不是随意确定的，必须满足特定要求，这些都是出于对处理器存取数据效率的考虑

对于各种基本数据类型来说，它的变量的内存地址值必须是其类型本身大小的整数倍，比如：

* 一个 int64 类型的变量的内存地址，应该能被 int64 类型自身的大小，也就是 8 整除
* 一个 uint16 类型的变量的内存地址，应该能被 uint16 类型自身的大小，也就是 2 整除

对于结构体而言，它的变量的内存地址，只要是它最长字段长度与系统对齐系数两者之间较小的那个的整数倍就可以了

比如在64位对齐系数为8的系统上：

    type T struct {
        b byte   // sum = 1 <--这里的数字表示几byte

        i int64  // sum = 1 + 7 + 8 <--这里插入了“缝隙”7
        u uint16 // sum = 1 + 7 + 8 + 2
                 // sum = 1 + 7 + 8 + 6 <--这里插入了“缝隙”6，是为了对齐整个结构体
    }

这样一来，不同的字段排列顺序也会影响到“填充字节”的多少，从而影响到整个结构体大小

所以，你在日常定义结构体时，一定要注意结构体中字段顺序，尽量合理排序，降低结构体对内存空间的占用

可以通过空标识符来进行主动填充，比如 runtime 包中的 mstats 结构体定义：

    // $GOROOT/src/runtime/mstats.go
    type mstats struct {
        ... ...
        // Add an uint32 for even number of size classes to align below fields
        // to 64 bits for atomic operations on 32 bit platforms.
        _ [1 - _NumSizeClasses%2]uint32 // 这里做了主动填充

        last_gc_nanotime uint64 // last gc (monotonic time)
        last_heap_inuse  uint64 // heap_inuse at mark termination of the previous GC
        ... ...
    }
