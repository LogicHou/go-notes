# 02

## Go版本

Go 团队承诺对最新的两个 Go 稳定大版本提供支持

选择最新版本一般没什么问题，保守的也可以选择次新版本

但是现在比较推荐1.17以上的版本，因为这个版本对defer进行了比较大的优化，而且对go module的支持也越来越完善

## Go安装

各安装步骤可以参考官方文档

多版本安装可以通过两种方法

第一种是重新设置 PATH 环境变量

第二种是使用go get 命令，前提是需要已经安装过一个GO版本

    $go get golang.org/dl/go1.15.13

    $go1.15.13 download

    Downloaded   0.0% (    16384 / 121120420 bytes) ...
    Downloaded   1.8% (  2129904 / 121120420 bytes) ...
    Downloaded  84.9% (102792432 / 121120420 bytes) ...
    Downloaded 100.0% (121120420 / 121120420 bytes)
    Unpacking /root/sdk/go1.15.13/go1.15.13.linux-amd64.tar.gz ...
    Success. You may now run 'go1.15.13'

下面贴出一些环境变量设置

    tee -a $HOME/.bashrc <<'EOF'
    # Go envs
    export GOVERSION=go1.17.2 # Go 版本设置
    export GO_INSTALL_DIR=$HOME/go # Go 安装目录
    export GOROOT=$GO_INSTALL_DIR/$GOVERSION # GOROOT 设置
    export GOPATH=$WORKSPACE/golang # GOPATH 设置
    export PATH=$GOROOT/bin:$GOPATH/bin:$PATH # 将 Go 语言自带的和通过 go install 安装的二进制文件加入到 PATH 路径中
    export GO111MODULE="on" # 开启 Go moudles 特性
    export GOPROXY=https://goproxy.cn,direct # 安装 Go 模块时，代理服务器设置
    export GOPRIVATE=
    export GOSUMDB=off # 关闭校验 Go 依赖包的哈希值
    EOF

### 环境变量的含义

| 环境变量    | 含义                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GOROOT      | Go 语言编译工具、标准库等的安装路径                                                                                                                                                                                                                                                                                                                                                                                              |
| GOPATH      | Go 的工作目录，也就是编译后二进制文件的存放目录和 import 包时的搜索路径                                                                                                                                                                                                                                                                                                                                                          |
| GO111MODULE | 通过设置 on、off、auto 来控制是否开启 Go Modules 特性。<br />on 代表开启 Go modules 特性，这会让 Go 编译器忽略 $GOPATH和vendor文件夹，只根据go.mod下载依赖；<br/>off代表关闭Gomodules特性，这会让 Go编译器在 $GOPATH目录和vendor目录来查找依赖关系，也就是继续使用“GOPATH模式”。而auto在 Go1.14和之后的版本中是默认值；<br />auto代表，源码在 $GOPATH/src 下，并且没有包含 go.mod 则关闭 Go modules，其它情况下都开启 Go modules |
| GOPROXY     | Go 包下载代理服务器。众所周知的原因，在大陆的网络环境下是无法访问 golang.org 等 Google 网站的，但在日 常开发中使用的很多依赖包都要从 Google 的服务器上下载。为了解决无法加载依赖的问题，需要设置一个代理服 务器。以便我们能够使用 go get 下载 <br /><br />direct 作用：当 go 在抓取目标模块时，若遇见了 404 错误，那么就直接去目标模块的源头（比如 GitHub）去抓 取，而不再通过代理服务器                                         |
| GOPRIVATE   | 指定不走代理的 Go 包域名。go get 通过代理服务拉取私有仓库（内部仓库或者托管站点的私有仓库），而代理服 务不可能访问到私有仓库，会出现了 404 错误。go1.13 版本提供了一个方便的解决方案：GOPRIVATE 环境变 量，通过该变量，可以使得指定的包不通过代理下载，而是直接下载                                                                                                                                                              |
| GOSUMDB     | GOSUMDB的值是一个web服务器，默认值是sum.golang.org, 该服务 月                                                                                                                                                                                                                                                                                                                                                                    |

## 程序结构

### 文件名

多个单词直接连接而不是使用分隔符，比如下划线

    helloworld.go  ⭕
    hello_world.go ❌

### 结构
    
    package main // 定义了一个包，main 包在 Go 中是一个特殊的包，整个 Go 程序中仅允许存在一个名为 main 的包

    import "fmt" // fmt代表的是包的导入路径（Import），它表示的是标准库下的 fmt 目录，整个 import 声明语句的含义是导入标准库 fmt 目录下的包；

    func main() { // 函数体开始
        fmt.Println("hello, world") // 这里的“fmt”代表的则是包名，首字母大写的函数名表示是导出的对包外可见，如果小写则只能在包内可见
    } // 函数体结束

标准 Go 代码风格使用 Tab 而不是空格来实现缩进的、

main包不能被导入

GO 使用 UTF-8 标准的字符编码方式

语句的结尾不需分号，Go 编译器会自动插入这些被省略的分号

### 编译

    $go build main.go // 编译一个go程序
    $go run main.go // 直接运行 Go 源码文件，多用于开发调试阶段

### Go module 1.16版本默认包管理机制

Go Module 的核心是一个名为 go.mod 的文件

    $go mod init github.com/username/hellomodule // 初始化一个module，添加go.mod文件
    go: creating new go.mod: module github.com/username/hellomodule
    go: to add module requirements and sums:
      go mod tidy

    $cat go.mod
    module github.com/username/hellomodule // 声明 module 的路径

    go 1.16 // Go 版本指示符，表示这个 module 是在某个特定的 Go 版本的 module 语义的基础上编写的

一个 module 就是一个包的集合，这些包和 module 一起打版本、发布和分发。go.mod 所在的目录是 module 的根目录

module 隐含了一个命名空间的概念，module 下每个包的导入路径都是由 module path 和包所在子目录的名字结合在一起构成

    github.com/username/hellomodule // module主路径
    github.com/username/hellomodule/pkg/pkg1 // 子目录 pkg/pkg1 的导入路径

不需要手动添加各个引入的包，用**go mod tidy**命令即可

    $go mod tidy       
    go: downloading go.uber.org/zap v1.18.1
    go: downloading github.com/valyala/fasthttp v1.28.0
    go: downloading github.com/andybalholm/brotli v1.0.2
    ... ...

随后自动生成的go.sum文件，记录了 module 的直接依赖和间接依赖包的相关版本的 hash 值，用来校验本地包的真实性

在构建的时候，如果本地依赖包的 hash 值与 go.sum 文件中记录的不一致，就会被拒绝构建

## Go项目的标准布局

Go 1.4 版本删除了 Go 源码树中“src/pkg/xxx”中 pkg 这一层级目录而直接使用 src/xxx，同时引入了引入 internal 包机制

一个 Go 项目里的 internal 目录下的 Go 包，只可以被本项目内部的包导入

Go1.6 版本增加 vendor 目录，直到 Go 1.7 版本才真正在 vendor 下缓存了其依赖的外部包

Go 1.13 版本引入 go.mod 和 go.sum

### 一个 Go 项目通常分为可执行程序项目和库项目

#### 可执行程序项目的典型结构布局（推荐使用的布局）

    $tree -F exe-layout 
    exe-layout.
    ├── all.bash  <--第三方的构建工具的脚本文件
    ├── make.bash <--第三方的构建工具的脚本文件
    ├── cmd/  <--存放项目要编译构建的可执行文件所对应的 main 包的源码文件
    │   ├── app1/ <--多个可执行文件，每个可执行文件的 main 包可以单独放在一个子目录中
    │   │   └── main.go
    │   └── app2/
    │       └── main.go
    ├── go.mod <--包依赖管理配置文件
    ├── go.sum <--包依赖管理哈希校验文件
    ├── internal/ <--存放仅项目内部引用的 Go 包，这些包无法被项目之外引用
    │   ├── pkga/
    │   │   └── pkg_a.go
    │   └── pkgb/
    │       └── pkg_b.go
    ├── pkg1/ <--项目自身使用、对应main包可执行文件所依赖的库文件，同时这些目录下的包能被外部项目引用
    │   └── pkg1.go
    ├── pkg2/ <--同上
    │   └── pkg2.go
    └── vendor/ <--go build -mod=vendor 可以实现基于 vendor 的构建，这是一个可选目录，为了兼容 Go 1.5 引入 vendor 构建模式而存在

main 包应该保持简洁，在其中会做一些命令行参数解析、资源初始化、日志设施初始化、数据库连接初始化等工作，之后再将程序的执行权限交给更高级的执行控制对象

如果项目结构中存在版本管理的“分歧”，建议将项目拆分为多个项目（仓库），每个项目单独作为一个 module 进行单独的版本管理和演进

单个可执行程序构建项目布局

    $tree -F -L 1 single-exe-layout
    single-exe-layout
    ├── go.mod
    ├── internal/
    ├── main.go
    ├── pkg1/
    ├── pkg2/
    └── vendor/

#### Go 库项目的典型结构布局

    $tree -F lib-layout 
    lib-layout
    ├── go.mod
    ├── internal/
    │   ├── pkga/
    │   │   └── pkg_a.go
    │   └── pkgb/
    │       └── pkg_b.go
    ├── pkg1/
    │   └── pkg1.go
    └── pkg2/
        └── pkg2.go

单个包的库项目布局

    $tree -L 1 -F single-pkg-lib-layout
    single-pkg-lib-layout
    ├── feature1.go
    ├── feature2.go
    ├── go.mod
    └── internal/

## 构建模式

### GOPATH模式

编译器可以在本地 GOPATH 环境变量配置的路径下，搜寻 Go 程序依赖的第三方包

如果存在，就使用这个本地包进行编译；如果不存在，就会报编译错误。