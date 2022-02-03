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

