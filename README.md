# go-notes

Go 学习过程中记录的一些重要知识点

# 目录

## [01](page01.md)

* Go 语言哲学：简单、显式、组合、并发和面向工程 

## [02](page02.md)

* Go 版本
* Go 安装
* 程序结构
* Go 项目的标准布局
* 构建模式
* Go Module的常规操作
* Go程序的执行次序

## [03 变量与作用域](page03.md)

* 命名
* 变量声明
* 代码块与作用域

## [04 基本数据类型、常量](page04.md)

* 基本数据类型
* 常量

## [05 复合数据类型](page05.md)

* 数组
* 切片
* map
* 结构体

## [06 控制结构](page06.md)

* if 语句
* for(for range) 循环
* switch 语句

## [07 函数](page07.md)

* 函数的声明
* 函数参数
* 函数是“一等公民”
* 结合多返回值进行错误处理
* 如何让函数更简洁健壮
* 语言中的异常：panic
* 使用 defer 简化函数实现

## [08 方法](page08.md)

* receiver 参数的相关约束
* 方法的本质
* 方法集合
* 如何选择receiver类型
* 使用类型嵌入模拟实现“继承”

## [09 接口](page09.md)

* 尽量定义“小接口”
* 接口的静态特性与动态特性
* 接口类型的装箱（boxing）原理
* Go 接口的应用模式或惯例

## [10 并发](page10.md)

* 概述
* Go 的并发方案：goroutine
  * goroutine 的基本用法
  * goroutine 的退出
  * goroutine 间的通信
  * Goroutine 调度器

## [11 并发原语](page11.md)

* channel
