## Go面试必考题目之method篇

原创：Joy  [Go后端干货]()   *2019-05-22*

​    在Go的类方法中，分为值接收者方法和指针接收者方法， 对于刚开始接触Go的同学来说，有时对Go的方法会感到困惑。下面我们结合题目来学习Go的方法。

​    为了方便叙述，下文描述的值接收者方法简写为值方法，指针接收者方法简写为指针方法。

​    

​    下面代码中，哪段编号的代码会报错？具体报什么错误？

```go
type Animal interface {
    Bark()
}

type Dog struct {
}

func (d Dog) Bark() {
    fmt.Println("dog")
}

type Cat struct {
}

func (c *Cat) Bark() {
    fmt.Println("cat")
}

func Bark(a Animal) {
    a.Bark()
}

func getDog() Dog {
    return Dog{}
}

func getCat() Cat {
    return Cat{}
}

func main() {
    dp := &Dog{}
    d := Dog{}
    dp.Bark() // (1)
    d.Bark()  // (2)
    Bark(dp)  // (3)
    Bark(d)   // (4)

    cp := &Cat{}
    c := Cat{}
    cp.Bark() // (5)
    c.Bark()  // (6)
    Bark(cp)  // (7)
    Bark(c)   // (8)
    
    getDog().Bark() // (9)
    getCat().Bark() // (10)
}
```

​    抛砖引玉，让我们学习完再来作答。



**值方法和指针方法**

------

​    我们来看看值方法的声明。

```go
type Dog struct {
}

func (d Dog) Bark() {
    fmt.Println("dog")
}
```

​    上面代码中，方法Bark的接收者是值类型，那么这就是一个值接收者的方法。

​    下面再看看指针接收者的方法。

```go
type Cat struct {
}

func (c *Cat) Bark() {
    fmt.Println("cat")
}
```

**类的方法集合**

------

这个在Go文档里有定义：

- 对于类型T，它的方法集合是所有接收者为T的方法。
- 对于类型*T，它的方法集合是所有接收者为*T和T的方法。



| Values | Method Sets      |
| ------ | ---------------- |
| T      | (t T)            |
| *T     | (t T) and (t *T) |

​    

**方法的调用者**

------

​    **指针\*T接收者方法：** 只有指针类型\*T才能调用，但其实值T类型也能调用，为什么呢？因为当使用值调用t.Call()时，Go会转换成(&t).Call()，也就是说最后调用的还是接收者为指针\*T的方法。

​    但要注意t是要能取地址才能这么调用，比如下面这种情况就不行：

```go
func getUser() User {
    return User{}
}

...

getUser().SayWat()
// 编译错误：
// cannot call pointer method on aUser()
// cannot take the address of aUser()
```

​    **值T接收者方法：** 指针类型\*T和值T类型都能调用。



| Methods Receivers | Values   |
| ----------------- | -------- |
| (t T)             | T and *T |
| (t *T)            | *T       |

​    使用接收者为\*T的方法实现一个接口，那么只有那个类型的指针\*T实现了对应的接口。  

​    如果使用接收者为T的方法实现一个接口，那么这个类型的值T和指针\*T都实现了对应的接口。



**声明建议**

------

​    在给类声明方法时，方法接收者的类型要统一，最好不要同时声明接收者为值和指针的方法，这样容易混淆而不清楚到底实现了哪些接口。

​    下面我们来看看哪种类型适合声明接收者为值或指针的方法。



**指针接收者方法**

​    下面这2种情况请务必声明指针接收者方法：

​    1. 方法中需要对接收者进行修改的。

​    2. 类中包含sync.Mutex或类似锁的变量，因为它们不允许值拷贝。

​    

​    下面这2种情况也建议声明指针接收者方法：

​    1. 类成员很多的，或者大数组，使用指针接收者效率更高。

​    2. 如果拿不准，那也声明接收者为指针的方法吧。



**值接收者方法**

​    下面这些情况建议使用值接收者方法：

​    1. 类型为map，func，channel。

​    2. 一些基本的类型，如int，string。

​    3. 一些小数组，或小结构体并且不需要修改接收者的。



**题目解析**

------

​    

```go
type Animal interface {
    Bark()
}

type Dog struct {
}

func (d Dog) Bark() {
    fmt.Println("dog")
}

type Cat struct {
}

func (c *Cat) Bark() {
    fmt.Println("cat")
}

func Bark(a Animal) {
    a.Bark()
}

func getDog() Dog {
    return Dog{}
}

func getCat() Cat {
    return Cat{}
}

func main() {
    dp := &Dog{}
    d := Dog{}
    dp.Bark() // (1) 通过
    d.Bark()  // (2) 通过
    Bark(dp)
    // (3) 通过，上面说了类型*Dog的方法集合包含接收者为*Dog和Dog的方法
    Bark(d)   // (4) 通过

    cp := &Cat{}
    c := Cat{}
    cp.Bark() // (5) 通过
    c.Bark()  // (6) 通过
    Bark(cp)  // (7) 通过
    Bark(c)
    // (8) 编译错误，值类型Cat的方法集合只包含接收者为Cat的方法
    // 所以T并没有实现Animal接口
    
    getDog().Bark() // (9) 通过
    getCat().Bark()
    // (10) 编译错误，
    // 上面说了，getCat()是不可地址的
    // 所以不能调用接收者为*Cat的方法
}
```



**总结**

------

1. 理清类型的方法集合。

2. 理清接收者方法的调用范围。 



**参考文献**

------

1. 《Pointer vs Value receiver》

https://yourbasic.org/golang/pointer-vs-value-receiver/

2. 《Method sets》

https://golang.org/ref/spec#Method_sets

3. https://stackoverflow.com/questions/19433050/go-methods-sets-calling-method-for-pointer-type-t-with-receiver-t?rq=1



**感谢阅读，欢迎大家指正，留言交流~**