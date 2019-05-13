## Go面试必考题目之defer篇

原创： Joy Go后端干货  *2019-05-13*

**下面程序分别输出什么?**

```go
func f1() {
    for i := 0; i < 5; i++ {
	defer fmt.Println(i)
    }
}

func f2() {
    for i := 0; i < 5; i++ {
	defer func() {
	    fmt.Println(i)
	}()
    }
}

func f3() {
    for i := 0; i < 5; i++ {
	defer func(n int) {
	    fmt.Println(n)
	}(i)
    }
}

func f4() int {
    t := 5
    defer func() {
	t++
    }()
    return t
}

func f5() (r int) {
    defer func() {
	r++
    }()
    return 0
}

func f6() (r int) {
    t := 5
    defer func() {
	t = t + 5
    }()
    return t
}

func f7() (r int) {
    defer func(r int) {
	r = r + 5
    }(r)
    return 1
}
```

​    同样的，我们先不急着作答，我们先来看看defer的讲解。



**defer是什么及用途**

------

- **defer是什么?**

​    defer 就像它的字面意思一样，就是延迟执行的意思，但是需要注意的是 defer 只能作用于函数，像变量赋值defer i = 10这种编译是会报错的。

- **defer函数的执行顺序**

​    被 defer 的函数会放入一个栈中，所以是先进后出的执行顺序，而被 defer 的函数在 return 之后执行。

- **清理释放资源**

​    当我们打开一个文件时，用完之后我们需要 close 这个文件，否则会导致文件描述符泄露，不用 defer 的代码一般是这样写的：

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```

​    上面代码文件都close了，但这里有一个问题：如果os.Open(srcName)成功了，然后在os.Create(dstName)发错误了，这时的 src 却没有 close ，这样就会导致src的文件描述符泄露了。 

​    修复这个问题也很简单，我们在os.Create(dstName)发生错误的时候，将src close掉就行了。但是问题又来了如果是多个文件同时打开，那这段代码将会非常的臃肿，而且会很容易的漏掉一些文件的 close。

​    使用defer可以完美的解决这个问题。

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

​    上面代码中，只需在文件操作成功的时候调用 defer close 就可以了，而且文件的 open 和 close 放在一起也不容易漏掉。



- **执行recover**

​    被 defer 的函数在 return 之后执行，这个时机点正好可以捕获函数抛出的 panic，因而defer 的另一个重要用途就是执行 recover ，而 recover 也只有在 defer 中才会起作用。

```go
func test() {
    defer func() {
        if ok := recover(); ok != nil {
            fmt.Println("recover")
        }
    }()
    panic("error")
}
```

​    另外需要注意的是，recover 要放在 panic 点的前面，一般放在函数的起始的位置就可以了。



**defer与return的关系**

------

​    Go 的函数返回值是通过堆栈返回的，这也是实现了多返回值的方法。看以下代码：

```go
// foo.go
package main

func foo() (int, int) {
   i := 1
   j := 2
   return i, j
}

func main() {
    foo()
}
```

生成的汇编代码如下：

```shell
$ go build -gcflags '-l' -o foo foo.go
$ go tool objdump -s "main\.foo" foo
TEXT main.foo(SB) /Users/kltao/code/go/src/example/foo.go
  bar.go:6    0x104ea70    48c744240801000000  MOVQ $0x1, 0x8(SP)
  bar.go:6    0x104ea79    48c744241002000000  MOVQ $0x2, 0x10(SP)
  bar.go:6    0x104ea82    c3      RET
```

**也就是说 return 语句不是原子操作，它被拆成了两步**

```go
rval = xxx // 返回值赋值给rval
ret // 函数返回
```

**而 defer 语句就是在这两条语句之间执行，也就是**

```go
rval = xxx // 返回值赋值给rval
defer_func  // 执行defer函数
ret // 函数返回
```

​    上面的题目中，还涉及到另外一个知识点，那就是闭包。

​    简单来说，Go 语言中的闭包就是在函数内引用函数体之外的数据，这样就会产生一种结果，虽然数据定义是在函数外，但是在函数内部操作数据也会对数据产生影响。看下面的例子：

```go
func foo() {
    i := 1
    func() {
        i++
    }
    fmt.Println(i) // 输出2
}
```

​    上面的 i 就是一个闭包引用，当匿名函数执行时，i 也会被修改。



**题目解析**

------

​    下面统一将rval称为函数最终return的变量值

```go
func f1() {
    for i := 0; i < 5; i++ {
        defer fmt.Println(i)
    }
}
// 因为defer的调用是先进后出的顺序
// 所以输出：5, 4, 3, 2, 1


func f2() {
    for i := 0; i < 5; i++ {
    	defer func() {
    	    fmt.Println(i)
    	}()
    }
}
// 上面说到，i是一个闭包引用
// 所以当执行defer时，i已经是5了
// 所以输出：5，5，5，5，5


func f3() {
    for i := 0; i < 5; i++ {
    	defer func(n int) {
    	    fmt.Println(n)
    	}(i)
    }
}
// Go的函数参数是值拷贝，所以这是普通的函数传值
// 所以输出：5，4，3，2，1


func f4() int {
    t := 5
    defer func() {
    	t++
    }()
    return t
}
// 注意：f4函数的返回值是没有声明变量的
// 所以t虽然是闭包引用，但返回值rval不是闭包引用
// 可以拆解为
// rval = t
// t++
// return rval
// 所以输出是5


func f5() (r int) {
    defer func() {
        r++
    }()
    return 0
}
// 注意：f5函数的返回值是有声明变量的
// 所以返回值r是闭包引用
// 可以拆解为
// r = 0
// rval = r
// r++
// return rval
// 所以输出：1


func f6() (r int) {
    t := 5
    defer func() {
    	t = t + 5
    }()
    return t
}
// 这里t虽然是闭包引用，但返回值r不是闭包引用
// 可以拆解为
// r = t
// rval = r
// t = t + 5
// return rval
// 所以输出：5


func f7() (r int) {
    defer func(r int) {
	r = r + 5
    }(r)
    return 1
}
// 因为匿名函数的参数也是r，所以相当于是
// 匿名函数的参数r = r + 5，不影响外部
// 所以输出：1
```

​    做这种defer的题，需要注意返回值是否为闭包引用。

   

**总结**

------

1. 谨记defer和return执行的顺序
2. 注意返回值是否为闭包引用



**参考文献**

------

1.《理解Go语言defer关键字的原理》https://draveness.me/golang-defer

2.《理解Go defer》https://sanyuesha.com/2017/07/23/go-defer/

3.《深入理解Go语言defer》https://mp.weixin.qq.com/s/e2t3CMUqtIcEq-OhbWy5Hw



**感谢阅读，欢迎大家指正，留言交流~**