## Go中的init函数

Joy   Go后端干货    *2019-06-13*

​    我们知道 `Go` 程序的入口是 `main` 函数，当 `main` 函数退出了，程序也就退出了。 `init` 函数在 `Go` 程序中也扮演着重要的角色。这篇文章将会介绍 `init` 函数的特性以及如何使用它们。



**`init` 函数的作用：**

- **变量初始化**

- **检查和修复程序状态**

- **运行前注册，例如 `decoder` ，`parser` 的注册**

- **运行只需计算一次的模块，像 `sync.once` 的作用**

- **其他**

  



**包初始化**

------

​    如果需要使用一个导入的包，首先要对这个包进行初始化，这一步在 `main` 函数执行之前，由 `runtime` 来完成，分以下步骤：

​    1. 初始化导入的包；

​    2. 初始化包作用域中的变量；

​    3. 执行包中的 `init` 函数。



​    **如果某个包被导入了多次，也只会执行一次包的初始化。**



**初始化顺序**

------

​    `Go` 一个包中可以包含很多文件，那么变量的初始化顺序与各个包的 `init` 函数执行顺序又是怎样的呢？

​    首先，`runtime` 的初始化依赖机制会启动，当初始化依赖机制计算完成后，就需要决定 `a.go` 和 `z.go` 中的变量谁先初始化，这取决于呈现给编译器的文件顺序，一般来说是按文件名的字典序，但是变量间或各个包间有依赖需要另外讨论。如果 `z.go` 先被传到 `build` 系统，那么 `z.go` 的变量初始化就比 `a.go` 先一步完成。

​    同一个包中，变量的初始化顺序是按文件名的字典序，但同时runtime也会解析变量间依赖关系，没有依赖的变量最先初始化，`init` 函数的执行顺序也同理。



​    来看下面按文件名字典序初始化的例子：


**`sandbox.go`**   

```go
package main

import "fmt"

var _ int64 = s()

func init() {
    fmt.Println("init in sandbox.go")
}

func s() int64 {
    fmt.Println("calling s() in sandbox.go")
    return 1
}

func main() {
    fmt.Println("main")
}
```

**`a.go`**

```go
package main

import "fmt"

var _ int64 = a()

func init() {
    fmt.Println("init in a.go")
}

func a() int64 {
    fmt.Println("calling a() in a.go")
    return 2
}
```

**`z.go`**

```go
package main

import "fmt"

var _ int64 = z()

func init() {
    fmt.Println("init in z.go")
}

func z() int64 {
    fmt.Println("calling z() in z.go")
    return 3
}
```

**程序输出：**

```shell
calling a() in a.go
calling s() in sandbox.go
calling z() in z.go
init in a.go
init in sandbox.go
init in z.go
main
```



下面是按依赖关系决定初始化顺序的例子。



**`pack.go`**

```go
package pack

import (
   "fmt"
   "test_util" // 引入test_util包
)

var Pack int = 6               

func init() {
   a := test_util.Util
   fmt.Println("init pack ", a)
}
```

**`test_util.go`**

```go
package test_util

import "fmt"

var Util int = 5

func init() {
   fmt.Println("init test_util")
}
```

**`main.go`**

```go
package main

import (
   "fmt"
   "pack"
   "test_util"                
)

func main() {
   fmt.Println(pack.Pack)
   fmt.Println(test_util.Util)
}
```

**输出：**

```shell
init test_util
init pack  5
6
5
```

​    **由于 `pack` 包的初始化依赖 `test_util`，因此运行时会先初始化 `test_util` 包再初始化 `pack` 包；**



**`init` 函数的特性**

------

​    `init` 函数不需要传入参数也没有返回值，而且init函数是不能被其他函数调用的。

```go
package main

import "fmt"

func init() {
    fmt.Println("init")
}

func main() {
    init()
}
```

​    上面的代码会报编译错误：`undefined: init`。



​    在一个文件中也可以有多个 `init` 函数，看下面代码。



**`sandbox.go`**

```go
package main

import "fmt"

func init() {
    fmt.Println("init 1")
}

func init() {
    fmt.Println("init 2")
}

func main() {
    fmt.Println("main")
}
```



**`utils.go`**

```go
package main

import "fmt"

func init() {
    fmt.Println("init 3")
}
```



**输出：**

```shell
init 1
init 2
init 3
main
```



​    `init` 函数的也广泛用在标准库中，比如 `math`，`bzip2`，`image`。

​    最常用的是初始化不能使用初始化表达式的变量，也就是不能在变量声明的时候初始化的变量，看以下例子。

```go
var square [10]int

func init() {
    for i := 0; i < 10; i++ {
        square[i] = i * i
    }
}
```



**只是为了执行 `init` 函数而导入包**

------

​    我们经常会在开源代码中见到有些导入的包中前面加了个下划线”_“，这表示只是想执行包中的 `init` 函数。

```go
import _ "image/png"
```

​    `image/png` 包里的 `init` 函数作用是向 `image` 包注册 `png` 图片的解码器，见 `src/image/png/reader.go`

```go
func init() {
	image.RegisterFormat("png", pngHeader, Decode, DecodeConfig)
}
```



**总结**

------

​    小心并且不要滥用 `init` 函数，因为对于复杂点的项目来说，`init` 函数的执行顺序难以捉摸。



**参考文献**

------

1.《init functions in Go》 https://medium.com/golangspec/init-functions-in-go-eac191b3860a

2.《五分钟理解golang的init函数》https://zhuanlan.zhihu.com/p/34211611

3.《When is the init() function run?》https://stackoverflow.com/questions/24790175/when-is-the-init-function-run



**感谢阅读，欢迎大家留言，分享，指正~**

