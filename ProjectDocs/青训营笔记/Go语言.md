## Go语言基础语法  

### Go开发环境的安装 

我习惯于vscode远程连接与linux服务器进行开发，所以可以在linux上安装go的环境，然后用vscode远程连接服务器，并安装vscode go环境所需要的插件。  

可看这篇文章进行安装：https://xuqilong.top/pages/3790c9/

### 基础语法  
基础语法不多说，感觉没有太多需要记录的，多用用就会了，简单记录一下我发现和C++不同的地方。  
1. 首先是变量用`:=`声明时不需要确定类型，go语言会根据上下文自动确定类型  
2. 循环语句没有`while`语句，可直接`for`关键字后加条件，作用与`while`相等  
3. `switch`的功能更加强大，`case`条件可以加表达式，而不仅仅是变量值，还有一点不同的是，`case`后无需加`break`，go语言自动加上`break`  
4. go语言存在slice数据类型，类似于C++的vector，可自动变长  
5. go语言有range关键字遍历数据，更加方便  
6. go语言中没有类这种东西，没有继承属性，只有结构体充当类，可以有成员函数和成员数据  

## Go语言依赖管理

### 并发编程
go语言并发编程非常简单，直接在想要并发的代码前加上`go`关键字即可，这样会开启goroutine，这是一种轻量级的执行单元，由Go运行时系统管理，无需操作系统内核管理。如想要某个函数并发执行，可直接`go func()`。  

goroutine之间的通信也非常简单，可直接使用Channel进行通信。Channel是一种通道，建立方式如下：  
```
ch := make(chan int)  // 建立一个无缓冲的int通道
ch := make(chan int, 2) // 建立一个缓冲大小为2的int通道
```
goroutine直接`ch <- 1`向通道中传入数据，而如果要取数据的话，可以`num <- ch`取出数据。需要注意的是，这两个操作无需加锁，因为channel已经内置了同步机制，每次只有一个goroutine访问通道。  
还有一点，当channel为空时，读数据会阻塞当前goroutine，反之，channel满时，取数据也会阻塞。  

除了通道，在并发编程中，锁是不可缺少的。Go语言同样提供锁机制，以下是使用方式：
```
lock sync.Mutex

lock.Lock()
/*dosomethins*/
lock.Unlock()
```

go语言还提供一种并发下的计时器，叫做WaitGroup，可帮助开发者在主goroutine等待其他子goroutine完成后再继续执行。  

WaitGroup的主要功能包括以下三个方法：

1. Add(delta int)：用于设置WaitGroup的计数器，表示需要等待的goroutine数量。delta表示要添加或减少的计数器值。  

2. Done()：用于减少WaitGroup的计数器值，表示一个goroutine已完成。一般在子goroutine的结束处调用Done()方法。  

3. Wait()：阻塞主goroutine，直到WaitGroup的计数器归零。主goroutine会在调用Wait()方法后等待所有的goroutine完成。  

课堂里的例子很好地展示它的作用:  
```
func ManyGoWait() {
    var wg sync.WaitGroup
    wg.Add(5);
    for i:=0; i<5; i++ {
        go func(j int) {
            defer wg.Done()
            fmt.Printf("%v ", j)
        }(i)
    }

    wg.Wait();
    fmt.Printf("Done\n");
    // 打印结果：4 0 1 2 3 Done
}
```
### 依赖管理

* GOPATH：Go支持的环境变量。项目目录结构为：src存放Go项目的源码；pkg存放编译的中间产物；bin存放Go项目编译生成的二进制文件。Go get [name] 命令可下载最近版本的包到src目录下。但无法实现包的多版本控制。  

* Go Vendor：项目目录下增加vendor文件，所有依赖包的副本放在vendor文件里。通过引入依赖的副本，解决多个项目需要同一个包依赖不同版本的冲突问题。假如项目两处依赖包的两个不同冲突版本，无法控制依赖的版本，会出现冲突，导致编译错误。  

* Go Module：通过go.mod文件管理依赖包版本，go get/go mod指令工具管理依赖包。  

### 测试

* 单元测试：所有测试文件以_test.go结尾，函数名为`func TestXxx(*Tesring.T)`，初始化逻辑放在TestMain中，运行`go test -run TestXxx`  

    怎么衡量项目的测试水准？**代码覆盖率**，一般覆盖率50%-60%  

* Mock测试：存放持久依赖的情况下，使用Mock测试，防止测试修改持久文件。  

* 基准测试：测试一段程序的运行性能和耗费CPU的程度。  

### 项目实践  

Go web项目结构通常分为数据层，逻辑层，视图层，数据层负责与持久化数据的增删查改，逻辑层负责核心业务逻辑输出，视图层处理和外部的交互逻辑，这同JAVA web相差不多。  

组件工具使用有gin和Go mod。gin是一个HTTP框架，简化了web开发的步骤，通知支持多种便捷的功能，如中间件插入等，大部分Go web开发皆可用gin。可看https://github.com/gin-gonic/gin。  







