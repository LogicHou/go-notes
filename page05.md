# 05 复合数据类型

## 定长数组

* 长度固定
* 类型同构

由元素的类型和数组长度（元素的个数）构成了 数组类型变量的声明：

    var arr [N]T
    # N代表长度，必须在声明时提供
    # T数组类型，可以为任意的 Go 原生类型或自定义类型(type)

N和T共同构成了数组的类型，如果有一个属性不同，就是两个不同的数组类型：

    func foo(arr [5]int) {}
    func main() {
        var arr1 [5]int
        var arr2 [6]int
        var arr3 [5]string

        foo(arr1) // ok
        foo(arr2) // 错误：[6]int与函数foo参数的类型[5]int不是同一数组类型
        foo(arr3) // 错误：[5]string与函数foo参数的类型[5]int不是同一数组类型
    }  

可以通过 len 函数获取一个数组的长度，通过 unsafe 包提供的 Sizeof 函数，获取一个数组变量的总大小：

    var arr = [6]int{1, 2, 3, 4, 5, 6}
    fmt.Println("数组长度：", len(arr))           // 6
    fmt.Println("数组大小：", unsafe.Sizeof(arr)) // 48 数组大小就是所有元素的大小之和

初始化：
    
    var arr1 [6]int // [0 0 0 0 0 0] 没有显式初始化，默认为元素值类型的零值
    
    var arr2 = [6]int {
        11, 12, 13, 14, 15, 16,
    } // [11 12 13 14 15 16] 进行了显示初始化

    var arr3 = [...]int { // “…” 表示由编译器自动计算出数组长度
        21, 22, 23,
    } // [21 22 23]
    fmt.Printf("%T\n", arr3) // [3]int

    var arr4 = [...]int{
        99: 39, // 将第100个元素(下标值为99)的值赋值为39，其余元素值均为0
    }
    fmt.Printf("%T\n", arr4) // [100]int

数组的访问，下标值从 0 开始：

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

## 变长切片 slice

声明：

    var nums = []int{1, 2, 3, 4, 5, 6}
    var sl1 []int     // 还没初始化，是nil值，底层没有分配内存空间
    var sl2 = []int{} // 初始化了，不是nil值，底层分配了内存空间，有地址

通过 len 函数获得切片类型变量的长度：

    fmt.Println(len(nums)) // 6

动态地向切片中添加元素：

    nums = append(nums, 7) // 切片变为[1 2 3 4 5 6 7]
    fmt.Println(len(nums)) // 7

### 切片类型的内部实现

    type slice struct {
        array unsafe.Pointer // array: 是指向底层数组的指针
        len   int            //   len: 是切片的长度，即切片中当前元素的个数
        cap   int            //   cap: 是底层数组的长度，也是切片的最大容量，cap 值永远大于等于 len 值
    }

### 通过 make 函数创建切片，并指定底层数组的长度

    sl := make([]byte, 6, 10) // 其中10为cap值，即底层数组长度，6为切片的初始长度

    sl := make([]byte, 6) // cap = len = 6 没有指定 cap 参数，那么底层数组长度 cap 就等于 len

### 用 array[low : high : max]语法基于一个已存在的数组创建切片，被称为数组的切片化

    arr := [10]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    sl := arr[3:7:9] // [4，5，6，7]
        
    sl[0] += 10
    fmt.Println("arr[3] =", arr[3]) // 14 改变了底层数组的值

* 起始元素从 low 所标识的下标值开始
* 切片的长度（len）是 high - low
* 容量是 max - low
* 切片 sl 的底层数组就是数组 arr，对切片 sl 中元素的修改将直接影响数组 arr 变量
* 切片之于数组就像是文件描述符之于文件，传递的并不是数组本身，而是一个大小固定的数组的“描述符”，开销小
* 在进行数组切片化的时候，通常省略 max，而 max 的默认值为数组的长度
* 针对一个已存在的数组，可以建立多个操作数组的切片，这些切片共享同一底层数组，对底层数组的操作也同样会反映到其他切片中

### 基于切片创建切片

原理与上面的一样

### 切片的动态扩容

通过 append 操作向切片追加数据的时候，如果这时切片的 len 值和 cap 值是相等的，则表示底层数组已经没有空闲空间再来存储追加的值，Go 运行时就会对这个切片做扩容操作，来保证切片始终能存储下追加的新值：

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
    fmt.Printf("after append 24, slice(len=%d, cap=%d): %v\n", len(s), cap(s), s)
    s = append(s, 25)
    fmt.Println("after append 25, array:", u)
    fmt.Printf("after append 25, slice(len=%d, cap=%d): %v\n", len(s), cap(s), s)
    s = append(s, 26)
    fmt.Println("after append 26, array:", u)
    fmt.Printf("after append 26, slice(len=%d, cap=%d): %v\n", len(s), cap(s), s)

    s[0] = 22
    fmt.Println("after reassign 1st elem of slice, array:", u)
    fmt.Printf("after reassign 1st elem of slice, slice(len=%d, cap=%d): %v\n", len(s), cap(s), s)

得到结果：
    
    array: [11 12 13 14 15]
    slice(len=2, cap=4): [12 13]
    after append 24, array: [11 12 13 24 15]
    after append 24, slice(len=3, cap=4): [12 13 24]
    after append 25, array: [11 12 13 24 25]
    after append 25, slice(len=4, cap=4): [12 13 24 25]
    after append 26, array: [11 12 13 24 25]
    after append 26, slice(len=5, cap=8): [12 13 24 25 26]
    after reassign 1st elem of slice, array: [11 12 13 24 25]
    after reassign 1st elem of slice, slice(len=5, cap=8): [22 13 24 25 26]

## map 类型

表示一组无序的键值对，集合中每个 key 都是唯一的，由 key 类型与 value 类型组成：

    map[key_type]value_type

key 与 value 的类型可相同或不同：

    map[string]string // key与value元素的类型相同
    map[int]string    // key与value元素的类型不同

map 类型对 value 的类型没有限制

但是为了保证 key 的唯一性，**key 的类型必须支持“==”和“!=”两种比较操作符**，有着严格的要求

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

所以函数类型、map 类型自身，以及切片类型是不能作为 map 的 key 类型的

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

所以必须对 map 类型变量进行显式初始化后才能使用

### 初始化

#### 使用复合字面值初始化 map 类型变量

    m := map[int]string{} // 此时变量 m 中没有任何键值对，但也不等同于初值为 nil 的 map 变量
    
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

#### 使用 make 进行显式初始化

    m1 := make(map[int]string) // 未指定初始容量
    m2 := make(map[int]string, 8) // 指定初始容量为8

map 不受限于它的初始容量值，当元素数量超过初始容量后，也会自动进行扩容

#### map 的基本操作

##### 插入新键值对

首先插入的对象必须是非 nil 的 map 类型变量，就可以在其中插入符合 map 类型定义的任意新键值对，直接赋值即可：

    m := make(map[int]string)
    m[1] = "value1"
    m[2] = "value2"
    m[3] = "value3"

除非系统内存耗尽，插入总是成功的，插入的时候如果 key 已经存在，会用新值覆盖旧值：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    m["key1"] = 11 // 11会覆盖掉"key1"对应的旧值1
    m["key3"] = 3  // 此时m为map[key1:11 key2:2 key3:3]

##### 获取键值对数量

通过 len 函数：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    fmt.Println(len(m)) // 2
    m["key3"] = 3  
    fmt.Println(len(m)) // 3

和切片类型不同，不能对 map 类型变量调用 cap 函数

##### 查找和数据读取

尝试获取一个键对应的值的时候，如果这个键在 map 中并不存在，会得到这个 value 元素类型的零值：

    m := make(map[string]int)
    v := m["key1"] // 如果 key1 不存在，就会被赋予 int 的零值，也就是0

使用“comma ok”惯用法对 map 进行键查找和键值读取操作：

    m := make(map[string]int)
    v, ok := m["key1"]
    if !ok {
        // "key1"不在map中
    }
    // "key1"在map中，v将被赋予"key1"键对应的value

    
    m := make(map[string]int)
    _, ok := m["key1"] // 用 _ 忽略可能返回的 value
    ... ...

##### 删除数据

使用内置函数 delete 来从 map 中删除数据：

    m := map[string]int {
      "key1" : 1,
      "key2" : 2,
    }

    fmt.Println(m) // map[key1:1 key2:2]
    delete(m, "key2") // 删除"key2"
    fmt.Println(m) // map[key1:1]

delete 函数是从 map 中删除键的唯一方法

即便传给 delete 的键在 map 中并不存在，delete 函数的执行也不会失败，更不会抛出运行时的异常

##### 遍历 map 中的键值数据

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

只关心每次迭代返回的键所对应的 value：

    for _, v := range m {
      // 使用v
    }

关于 map 一个很重要的问题就是，**对同一 map 做多次遍历的时候，每次遍历元素的次序都不相同**

这是一个坑，所以 Go **程序逻辑千万不要依赖遍历 map 所得到的的元素次序**

#### map 变量的传递开销

和切片类型一样，map 也是引用类型，实质上传递的也只是一个“描述符”，传递的开销是固定的，而且也很小

而且以描述符传递的类型(map slice 引用 )，在函数或者方法内部中的修改对外部也是可以的

#### map 的内部实现

Go 运行时使用一张哈希表来实现抽象的 map 类型，实现了包括查找、插入、删除等

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

#### map 与并发

map 实例不是并发写安全的，也不支持并发读写，如果对 map 实例进行并发读写，程序运行时就会抛出异常

如果仅仅是进行并发读，map 是没有问题的

并发写安全的 sync.Map 类型(Go version >= 1.9)，可以用来在并发读写的场景下替换掉 map

注意，因为 map 可以自动扩容，map 中数据元素的 value 位置可能在这一过程中发生变化

所以 Go 不允许获取 map 中 value 的地址，这个约束在编译期间就生效，如果对 value 取地址就会报编译错误：

    p := &m[key]  // cannot take the address of m[key]
    fmt.Println(p)
