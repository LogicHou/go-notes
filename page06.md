# 01 控制结构

“程序 = 数据结构 + 算法”

再复杂的算法都可以通过顺序、分支和循环这三种基本的控制结构构造出来

分支结构，Go 提供了 if 和 switch-case 两种语句形式

循环结构，Go 只保留了 for 这一种循环语句形式，且只有 for 这一种循环语句

## Go 的 if 语句

根据**布尔表达式**的值，在两个分支中选择一个执行

if 语句的布尔表达式整体不需要用括号包裹，if 关键字后面的条件判断表达式的求值结果必须是布尔类型，即要么是 true，要么是 false：

    if boolean_expression {
        // 新分支
    }

    // 原分支

可以用多个逻辑操作符连接起多个条件判断表达式：

    if (runtime.GOOS == "linux") && (runtime.GOARCH == "amd64") &&
        (runtime.Compiler != "gccgo") {
        println("we are using standard go compiler on linux os for amd64")
    }

| 逻辑操作符 | 含义   |
| ---------- | ------ |
| &&         | 逻辑与 |
| \\|\\|     | 逻辑或 |
| ！         | 逻辑非 |

操作符优先级：

| 优先级（从高到低） | 操作符列表             |
| ------------------ | ---------------------- |
| 5                  | *、/、%、<<、>>、&、&^ |
| 4                  | +、-                   |
| 3                  | !=、==、<、<=、>、>=   |
| 2                  | &&                     |
| 2                  | \\|\\|                 |

二分支结构：

    if boolean_expression {
      // 分支1
    } else {
      // 分支2
    }

多分支结构：

    if boolean_expression1 {
      // 分支1
    } else if boolean_expression2 {
      // 分支2

    ... ...

    } else if boolean_expressionN {
      // 分支N
    } else {
      // 分支N+1
    }

支持声明 if 语句的自用变量，只可以在 if 语句的代码块范围内使用，是 Go 语言的一个惯用法：

    func main() {
        if a, c := f(), h(); a > 0 { <--a，c 的作用域开始
            println(a)
        } else if b := f(); b > 0 { <--b的作用域开始
            println(a, b)
        } else {
            println(a, b, c)
        } <--a，b，c 作用域结束
    }

### if 语句的“快乐路径”原则：

    func doSomething() error {
      if errorCondition1 {
        // some error logic
        ... ...
        return err1
      }
      
      // some success logic
      ... ...

      if errorCondition2 {
        // some error logic
        ... ...
        return err2
      }

      // some success logic
      ... ...
      return nil
    }

上面代码的特点展现了快乐路劲原则的特点：

* 没有使用 else 分支，失败就立即返回
* “成功”逻辑始终“居左”并延续到函数结尾，没有被嵌入到 if 的布尔表达式为 true 的代码分支中
* 整个代码段布局扁平，没有深度的缩进


所谓**快乐路径**原则，就是：

* 仅使用单分支控制结构
* 当布尔表达式求值为 false 时，也就是出现错误时，在单分支中快速返回
* 正常逻辑在代码布局上始终“靠左”，可以从上到下一眼看到该函数正常逻辑的全貌
* 函数执行到最后一行代表一种成功状态。

如何对不符合“快乐路径”的 if 代码，进行重构：

* 尝试将“正常逻辑”提取出来，放到“快乐路径”中
* 如果无法做到上一点，很可能是函数内的逻辑过于复杂，可以将深度缩进到 else 分支中的代码析出到一个**函数**中，再对原函数实施“快乐路径”原则

### if 代码优化技巧：把概率最大的分支放到最上面

    func foo() {
        if boolean_expression1 { <--概率最大的

        } else if boolean_expression2 { <--概率比较大的

        } else if boolean_expression3 { <--概率一般大的

        } else { <--概率最小的

        }
    }

## for 语句

### for 语句经典形式

由四个部分组成：

* 循环前置语句，比如 i := 0;
* 条件判断表达式，比如 i < 10;
* 循环体
* 循环后置语句，比如 i++

    var sum int
    for i := 0; i < 10; i++ {
        sum += i
    }
    println(sum)

另外 Go 语言的 for 循环支持声明多循环变量，并且可以应用在循环体以及判断条件中

    for i, j, k := 0, 1, 2; (i < 20) && (j < 10) && (k < 30); i, j, k = i+1, j+1, k+5 {
        sum += (i + j + k)
        println(sum)
    }

Go 的 for 循环除了循环体部分，其余的三个部分都是可选的：

    for i := 0; i < 10; { <--省略了循环后置语句
        i++
    }
    
    i := 0
    for ; i < 10; i++{ <--省略了循环前置语句
        println(i)
    }  

    i := 0
    for ; i < 10; { <--同时省略了循环前置和后置语句，但是分号需要保留不能省略
        println(i)
        i++
    }  
    
    i := 0
    for i < 10 { <--只有当循环前置与后置语句都省略掉，仅保留循环判断条件表达式时，此时可以省略分号
        println(i)
        i++
    }

    for { <--当循环判断条件表达式的求值结果始终为 true 时，可以省略掉全部
      // 循环体代码
    }

    // 等价于上面的写法，就是通常所说的“无限循环”
    for true {
      // 循环体代码
    }

    // 或者
    for ; ; {
      // 循环体代码
    }

### for range 循环形式

    var sl = []int{1, 2, 3, 4, 5}
    for i := 0; i < len(sl); i++ {
        fmt.Printf("sl[%d] = %d\n", i, sl[i])
    }

    for i, v := range sl {
        fmt.Printf("sl[%d] = %d\n", i, v)
    }

    // 不关心元素值时
    for i := range sl {
      // ... 
    }
    
    // 不关心元素下标时
    for _, v := range sl {
      // ... 
    }

    // 不关心下标值，也不关心元素值
    for _, _ = range sl {
      // ... 
    }
    
    for range sl {
      // ... 
    }

#### string 类型

每次循环得到的 v 值是一个 Unicode 字符码点，也就是 rune 类型值，而不是一个字节

返回的第一个值 i 为该 Unicode 字符码点的内存编码（UTF-8）的第一个字节在字符串内存序列中的位置

    var s = "中国人"
    for i, v := range s {
        fmt.Printf("%d %s 0x%x\n", i, string(v), v)
    }
        
    0 中 0x4e2d
    3 国 0x56fd
    6 人 0x4eba

#### map

要对 map 进行循环操作，for range 是唯一的方法

每次循环，循环变量 k 和 v 分别会被赋值为 map 键值对集合中一个元素的 key 值和 value 值

    var m = map[string]int {
      "Rob" : 67,
        "Russ" : 39,
        "John" : 29,
    }

    for k, v := range m {
        println(k, v)
    }
    
    John 29
    Rob 67
    Russ 39

#### channel

channel 是 Go 语言提供的并发设计的原语，用于多个 Goroutine 之间的通信

for range 会尝试从 channel 中读取数据：

    var c = make(chan int)
    for v := range c {
      // ... 
    }

for range 每次从 channel 中读取一个元素后，会把它赋值给循环变量 v，并进入循环体

当 channel 中没有数据可读的时候，for range 循环会阻塞在对 channel 的读操作上

直到 channel 关闭时，for range 循环才会结束，这也是 for range 循环与 channel 配合时隐含的循环判断条件

### 带 label 的 continue 语句

普通的 continue 语句只会在最内层的循环进入下一次迭代：

    var sum int
    var sl = []int{1, 2, 3, 4, 5, 6}
    for i := 0; i < len(sl); i++ {
        if sl[i]%2 == 0 {
            // 忽略切片中值为偶数的元素
            continue
        }
        sum += sl[i]
    }
    println(sum) // 9

带 label 的 continue 语句，通常出现于嵌套循环语句中，被用于跳转到外层循环并继续执行外层循环语句的下一个迭代：

    func main() {
        var sl = [][]int{
            {1, 34, 26, 35, 78},
            {3, 45, 13, 24, 99},
            {101, 13, 38, 7, 127},
            {54, 27, 40, 83, 81},
        }

    outerloop:
        for i := 0; i < len(sl); i++ {
            for j := 0; j < len(sl[i]); j++ {
                if sl[i][j] == 13 {
                    fmt.Printf("found 13 at [%d, %d]\n", i, j)
                    continue outerloop <--中断内层 for 循环，回到外层 for 循环继续执行
                }
            }
        }
    }

### break 语句的使用

    func main() {
        var sl = []int{5, 19, 6, 3, 8, 12}
        var firstEven int = -1

        // 找出整型切片sl中的第一个偶数
        for i := 0; i < len(sl); i++ {
            if sl[i]%2 == 0 {
                firstEven = sl[i]
                break
            }
        }

        println(firstEven) // 6
    }

带 lable：

    var gold = 38

    func main() {
        var sl = [][]int{
            {1, 34, 26, 35, 78},
            {3, 45, 13, 24, 99},
            {101, 13, 38, 7, 127},
            {54, 27, 40, 83, 81},
        }

    outerloop:
        for i := 0; i < len(sl); i++ {
            for j := 0; j < len(sl[i]); j++ {
                if sl[i][j] == gold {
                    fmt.Printf("found gold at [%d, %d]\n", i, j)
                    break outerloop
                }
            }
        }
    }

### for 语句的常见“坑”与避坑方法

for 语句的常见“坑”点通常和 for range 这个“语法糖”有关

#### 循环变量的重用

    func main() {
        var m = []int{1, 2, 3, 4, 5}  
                
        for i, v := range m {
            go func() {
                time.Sleep(time.Second * 3)
                fmt.Println(i, v)
            }()
        }

        time.Sleep(time.Second * 10)
    }

    // 看起来会输出     // 实际上会输出
    0 1               4 5
    1 2               4 5
    2 3               4 5
    3 4               4 5
    4 5               4 5

不能简单地认为每次迭代都会重新声明两个新的变量 i 和 v

实际这些循环变量在 for range 语句中仅会被声明一次，且在每次迭代中都会被重用

上面代码等价转换后时这样的：

    func main() {
        var m = []int{1, 2, 3, 4, 5}  
                
        {
          i, v := 0, 0 // 其实只声明了一次
            for i, v = range m { // 循环变量 i 和 v 在每次迭代时会重用
                go func() {
                    time.Sleep(time.Second * 3)
                    fmt.Println(i, v) // i, v 值在整个循环过程中是重用的，仅有一份
                }() // Goroutine 执行完的时候 for range早就结束了，最终值 i = 4, v = 5
            }
        }

        time.Sleep(time.Second * 10)
    }

想要输出预想的效果，需要创建 Goroutine 时将参数与 i、v 的当时值进行绑定：

    func main() {
        var m = []int{1, 2, 3, 4, 5}

        for i, v := range m {
            go func(i, v int) {
                time.Sleep(time.Second * 3)
                fmt.Println(i, v)
            }(i, v)
        }

        time.Sleep(time.Second * 10)
    }

    // 输出结果，由于 Goroutine 的调度所决定，输出结果的行序可能不同
    0 1
    1 2
    2 3
    3 4
    4 5

#### 参与循环的是 range 表达式的副本

在 for range 语句中，range 后面接受的表达式的类型可以是数组、指向数组的指针、切片、字符串，还有 map 和 channel（需具有读权限）

**参与 for range 循环的是都是 range 表达式的副本**

遍历数组中的坑：

    func main() {
        var a = [5]int{1, 2, 3, 4, 5}
        var r [5]int

        fmt.Println("original a =", a) // 输出 [1 2 3 4 5]

        for i, v := range a { // 这里的 a 可以看成是a' 是a的一个副本，每次迭代的都是从数组 a 的值拷贝 a’中得到的元素
                              // a’是 Go 临时分配的连续字节序列，与 a 完全不是一块内存区域
            if i == 0 {
                a[1] = 12 // 改变了循坏外 a 的值，而不是循环内 a' 的值
                a[2] = 13
            }
            r[i] = v // 这里的 v 取自 a' 而不是 a 中的值，所以依次取得了 1，2，3，4，5
        }

        fmt.Println("after for range loop, r =", r) // 输出 [1 2 3 4 5]
        fmt.Println("after for range loop, a =", a) // 输出 [1 12 13 4 5]
    }

可以通过传入数组的引用 &a 来解决这个问题

再来看看切片：

    func main() {
        var a = [5]int{1, 2, 3, 4, 5}
        var r [5]int

        fmt.Println("original a =", a) // 输出 [1 2 3 4 5]

        for i, v := range a[:] {
            if i == 0 {
                a[1] = 12
                a[2] = 13
            }
            r[i] = v
        }

        fmt.Println("after for range loop, r =", r) // 输出 [1 12 13 4 5]
        fmt.Println("after for range loop, a =", a) // 输出 [1 12 13 4 5]
    }

切片没有问题，了解切片原理就应该明白

#### 遍历 map 中元素的随机性

如果在循环的过程中，对 map 进行了修改，这个结果和我们遍历 map 一样，具有随机性

在 map 循环过程中，当 counter 值为 0 时，我们删除了变量 m 中的一个元素的例子：

    var m = map[string]int{
        "tony": 21,
        "tom":  22,
        "jim":  23,
    }

    counter := 0
    for k, v := range m {
        if counter == 0 {
            delete(m, "tony")
        }
        counter++
        fmt.Println(k, v) // 当 k="tony" 作为第一个迭代的元素时，即使被进行了删除操作，也自然会被遍历到，从而输出一次
    }
    fmt.Println("counter is ", counter)

当 k="tony" 作为第一个迭代的元素时，将得到如下结果：

    tony 21
    tom 22
    jim 23
    counter is  3

否则：

    tom 22
    jim 23
    counter is  2

如果在针对 map 类型的循环体中，新创建了一个 map 元素项，那这项元素可能出现在后续循环中，也可能不出现：

    var m = map[string]int{
        "tony": 21,
        "tom":  22,
        "jim":  23,
    }

    counter := 0
    for k, v := range m {
        if counter == 0 {
            m["lucy"] = 24 // 是否访问到，视m["lucy"]=24这个键值对的插入位置而定，如果插入位置在最前面，遍历的时候就访问不到，如果插入位置靠后，在后面的迭代中就会被访问到
        }
        counter++
        fmt.Println(k, v)
    }
    fmt.Println("counter is ", counter)

会有两个结果：

    tony 21
    tom 22
    jim 23
    lucy 24
    counter is  4

或：

    tony 21
    tom 22
    jim 23
    counter is  3