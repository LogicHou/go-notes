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

