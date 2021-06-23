# Golang代码题

以下代码有什么问题，怎么解决？

```go
total, sum := 0, 0
for i := 1; i <= 10; i++ {
    sum += i
    go func() {
        total += i
    }()
}
fmt.Printf("total:%d sum %d", total, sum)
```

## 01 考点一

相信很多人应该一眼看出了其中的一个问题，那就是 i 使用的问题。

以下代码输出什么？

```go
for i := 1; i <= 10; i++ {
  go func() {
    fmt.Println(i)
  }()
}
time.Sleep(1e9)
```

相信很多人知道，输出 较多 的 11，而不是期望的输出 1 到 10。

怎么改进？你应该也知晓。

```go
for i := 1; i <= 10; i++ {
  go func(i int) {
    fmt.Println(i)
  }(i)
}
time.Sleep(1e9)
```

（当然这里的输出顺序是乱的，大家应该清楚）

## 02 考点二

该题的第二个考点：data race。因为存在多 goroutine 同时写 total 变量的问题，所以有数据竞争。可以加上 -race 参数验证：

```bash
$ go run -race main.go
==================
WARNING: DATA RACE
Read at 0x00c0001b4020 by goroutine 8:
  main.main.func1()
      /Users/xuxinhua/main.go:12 +0x57

Previous write at 0x00c0001b4020 by main goroutine:
  main.main()
      /Users/xuxinhua/main.go:9 +0x10b

Goroutine 8 (running) created at:
  main.main()
      /Users/xuxinhua/main.go:11 +0xe7
==================
```

这可以通过加锁的方式解决：

```go
var mutex sync.Mutex
total, sum := 0, 0
for i := 1; i <= 10; i++ {
  sum += i
  go func(i int) {
    mutex.Lock()
    total += i
    mutex.Unlock()
  }(i)
}
```

此外，也可以通过 atomic 包解决：（注意 total 的类型，因为 atomic.AddInt64 需要）

```go
var total int64
sum := 0
for i := 1; i <= 10; i++ {
  sum += i
  go func(i int) {
    atomic.AddInt64(&total, int64(i))
  }(i)
}
```

通过 -race 你验证，发现 data race 没了。

细心的你不知道发现没有，以上代码我故意把最后的 fmt 输出那一行去掉了，因为它用了 total 变量，避免它导致 data race。这引出考点三。

## 03 考点三

我上面都没有给完整的代码，因为经过上面两步，最终的结果还是不对的。从上面说的 fmt 输出代码去掉就说明还有问题。

初学 Go 应该遇到类似这样的问题，下面代码一般没有输出。

```go
package main

import "fmt"

func main() {
 go func() {
  fmt.Println("Hello World!")
 }()
}
```

原因是 main 函数先退出了，开启的 goroutine 根本没有机会执行。所以，常见的解决办法是在最后加一个 Sleep：

```go
package main

import "fmt"

func main() {
 go func() {
  fmt.Println("Hello World!")
 }()
  
  time.Sleep(1e9)
}
```

Sleep 会让 main goroutine 休眠，调度器调度其他 goroutine 运行。

回到开头的题目其实也存在这个问题，通过在 fmt 语句之前加上 Sleep，基本能得到正确的结果：

```go
var total int64
sum := 0
for i := 1; i <= 10; i++ {
    sum += i
    go func(i int) {
        atomic.AddInt64(&total, int64(i))
    }(i)
}
time.Sleep(1e9)

fmt.Printf("total:%d sum %d", total, sum)
```

但如果加上 -race 发现还是有问题：

```bash
$ go run -race main.go
==================
WARNING: DATA RACE
Read at 0x00c00001c0b0 by main goroutine:
  main.main()
      /Users/xuxinhua/main.go:20 +0xe4

Previous write at 0x00c00001c0b0 by goroutine 7:
  sync/atomic.AddInt64()
      /Users/xuxinhua/.go/current/src/runtime/race_amd64.s:276 +0xb
  main.main.func1()
      /Users/xuxinhua/main.go:15 +0x44

Goroutine 7 (finished) created at:
  main.main()
      /Users/xuxinhua/main.go:14 +0xa4
==================
total:55 sum 55Found 1 data race(s)
```

所以，这种方式是不靠谱的，这时正确的方式是使用 sync.WaitGroup。

```go
package main

import (
    "sync/atomic"
    "sync"
    "fmt"
)

func main() {
    var wg sync.WaitGroup
    var total int64
    sum := 0
    for i := 1; i <= 10; i++ {
        wg.Add(1)
        sum += i
        go func(i int) {
            defer wg.Done()
            atomic.AddInt64(&total, int64(i))
        }(i)
    }
    wg.Wait()

    fmt.Printf("total:%d sum %d", total, sum)
}
```

# 选择题

## const

Go 语言中可以省略类型说明符 [type]，因为编译器可以根据变量的值来推断其类型；

存储在常量中的数据类型只可以是布尔型、数字型（整数型、浮点型和复数）和字符串型。

常量是编译时就能确定的数据，error是对象数据类型，是一个指针，不是常量。errors.New("xxx") 要等到运行时才能确定，所以不满足。

Go中的常量通常是无类型的。但可以参与一些“有类型”的计算。Go语言的常量有个不同寻常之处。虽然一个常量可以有任意有一个确定的基础类型，例如int或float64，或者是类似time.Duration这样命名的基础类型，但是许多常量并没有一个明确的基础类型。编译器为这些没有明确的基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度。这里有六种未明确类型的常量类型，分别是无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串。

## bool

1. flag是bool类型，if flag==false写法不符合规范

## string

1. GO语言中字符串是不可变的，所以不能对字符串中某个字符单独赋值。
2. go语言中字符串是子UTF-8编码并存储的，它语言不定长的字节，所以它不支持下标操作，因为没一个下标操作代表的是固定长度的字节。

## chan

1. <- ch正确；<- ch 可以单独调用获取通道的（下一个）值，当前值会被丢弃，但是可以用来验证。<-chan int  像这样的只能接收值
2. ch <-错误；chan<- int  像这样的只能发送值
3. 无缓冲的channel是同步的，而有缓冲的channel是非同步的
4. nil channel代表channel未初始化，向未初始化的channel读写数据会造成永久阻塞
5. 给一个 nil channel 发送数据，造成永远阻塞
6. 从一个 nil channel 接收数据，造成永远阻塞
7. 给一个已经关闭的 channel 发送数据，引起 panic
8. 从一个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回一个零值

## slice

go语言编译器会自动在以标识符、数字字面量、字母字面量、字符串字面量、特定的关键字（break、continue、fallthrough和return）、增减操作符（++和--）、或者一个右括号、右方括号和右大括号（即)、]、}）结束的非空行的末尾自动加上分号。

```go
// 6是数字字面量，所以在6的后面会自动加上一个分号，导致编译出错
x := []int{
    1, 2, 3,
    4, 5, 6
}

// gofmt会自动把6后面的“,”去掉，关掉gofmt后测试，也能通过编译，正常运行
x := []int{1, 2, 3, 4, 5, 6, }
```

Make只用来创建slice,map,channel。 其中map使用前必须初始化。 append可直接动态扩容slice，而map不行。

```go
var s []int
s = append(s, 1)
// 可以正确运行
```



## select

1. select机制最大的一条限制就是每个case语句里必须是一个IO操作
2. select关键字的用法与switch语句不同，后面不需要带判断条件
3. switch 中的表达式是可选的，可以省略。如果省略表达式，则相当于 switch true，这种情况下会将每一个 case 的表达式的求值结果与 true 做比较，如果相等，则执行相应的代码。
4. 可以在一个 case 中包含多个表达式，每个表达式用逗号分隔。
5. switch语句条件表达式不是必须为常量或者整数

## map

1. map反序列化时json.unmarshal的入参必须为map的地址
2. 在函数调用中传递map，则子函数中对map元素的增加或者修改会导致父函数中map的修改
3. 可以使用内置函数delete删除map的元素
4. cap可用于chan，不支持map，map中使用len表示大小
5. cap的作用：array返回数组的元素个数，slice返回slice的最大容量， channel返回channel的buffer容量
6. map是引用对象，在子函数中对引用对象进行操作会影响到父函数中引用对象

## interface

1. 类实现接口时不需要导入接口所在的包
2. 只要两个接口拥有相同的方法列表（次序不同不要紧），那么它们就是等价的，可以相互赋值
3. 如果接口A的方法列表是接口B的方法列表的子集，那么接口B可以赋值给接口A
4. 接口查询是否成功，要在运行期才能够确定
5. interface{}可以初始化为nil

关于GetPodAction定义，下面赋值正确的是（）  

  ![img](https://uploadfiles.nowcoder.com/images/20171203/3367369_1512279121001_4196C83899FB2F0093E735B20B5B266F)

> A. var fragment Fragment = new(GetPodAction)
> B. var fragment Fragment = GetPodAction
> C. var fragment Fragment = &GetPodAction{}
> D. var fragment Fragment = GetPodAction{}

1. 使用结构体作为接收者实现接口，此时无论使用结构体初始化变量还是结构体指针初始化变量，都可以编译通过 
2. 使用结构体指针作为接收者实现接口，如果使用结构体指针初始化变量， 如果结构体初始化变量，则不能编译通过 

本例中，使用GetPodAction结构体作为接收者实现Fragment接口，则使用*GetPodAction类型初始化接口变量，编译通过并赋值成功，AC正确 

D肯定也正确。如果使用*GetPodAction类型作为接收者实现Fragment接口，如果使用GetPodAction结构体初始化接口变量，会报错

## for

1. for循环不支持以逗号为间隔的多个赋值语句，必须使用平行赋值的方式来初始化多个变量
2. for后面的语句中不能有逗号分割的语句，各个语句必须都是平等的，使用分号分割。for后面可以有无数多个分号

等号左边和右边含有多个表达式，就是平行赋值。 

由于Go没有逗号表达式，而++和--是语句而不是表达式，如果想在for中执行多个变量,需要使用平行赋值

```go
for i, j := 1, 10; i < j; i,j=i+1,j+1 {  //死循环
    fmt.Println(i)
}
```

而不能写成

```go
for i, j := 1, 10; i < j; i++,j++ {
    fmt.Println(i)
}
```

for的condition在每执行一次循环体时便会执行一次，因此在实际开发过程中需要注意不要让condition中计算简单而不是复杂。

```go
for i,j :=0,len(str); i<j ; i++ {
    fmt.Println(str[i])
}
```

而不要写成

```go
for i=0; i< len(str); i++ {  
    fmt.Println(str[i]) 
}
```

## init

1. init函数不可以被其他函数调用
2. main包中可以有init
3. init函数可以在任何包中有0个或1个或多个；
4. 首先初始化导入包的变量和常量，然后执行init函数，最后初始化本包的变量和常量，然后是init函数，最最后是main函数；
5. main函数只能在main包中有且只有一个，面包中也可以有0或1或多个init函数；
6. init函数和main函数都不能被显示调用；

## panic

当内置的panic()函数调用时，外围函数或方法的执行会立即终止。然后，任何延迟执行(defer)的函数或方法都会被调用，就像其外围函数正常返回一样。最后，调用返回到该外围函数的调用者，就像该外围调用函数或方法调用了panic()一样，因此该过程一直在调用栈中重复发生：函数停止执行，调用延迟执行函数等。当到达main()函数时不再有可以返回的调用者，因此这个过程会终止，并将包含传入原始panic()函数中的值的调用栈信息输出到os.Stderr。

panic需要等defer结束后才会向上传递。出现panic的时候，会先按照defer的后入先出的顺序执行，最后才会执行panic。协程会遍历所有调用栈上需要执行的defer，如果执行完这些defer没有遇到recover，会直接调用exit退出程序。

## CGO

CGO是调用C代码模块，静态库和动态库。CGO不支持C++，但是可以通过C来封装C++的方法实现CGO调用。CGO是C语言和Go语言之间的桥梁，原则上无法直接支持C++的类。CGO不支持C++语法的根本原因是C++至今为止还没有一个二进制接口规范(ABI)。CGO只支持C语言中值类型的数据类型，所以我们是无法直接使用C++的引用参数等特性的。

## 匿名变量

如果调用方调用了一个具有多返回值的方法，但是却不想关心其中的某个返回值，可以简单地用一个下划线“_”来跳过这个返回值，该下划线对应的变量叫匿名变量

## 多个参数

![img](https://uploadfiles.nowcoder.com/images/20171203/3367369_1512277319335_04CAB4983DCD2275AD7C5F8E58976A81)

```go
// ...会把int数组中元素转成int的多个参数，因此也是正确的
add([]int{1, 3, 7}...)
```

![img](https://uploadfiles.nowcoder.com/images/20190810/150238853_1565428069602_9088E622408688FD730D17144C11D7DD)

## 指针

1. 指针可以转化为JSON，channel、complex、函数不可以
2. go语言的指针不支持运算
3. 方法施加的对象不需要非得是指针，也不用非得叫this；方法施加的对象显式传递，没有被隐藏起来
4. 指针不是基础类型，是复合类型，也是引用类型

通过指针变量 p 访问其成员变量 name，下面语法正确的是（）

> 正确答案: A B  你的答案: A B C (错误)

```
p.name
(*p).name
(&p).name
p->name
```

GO语言中访问成员变量的方式只有 **.** 号（因为->是用于通道的操作符，所以go语言中指针不支持->操作符），并且GO语言足够智能，能够自动解引用，但智能也是有限的，只能解一次引用，指针的指针还得自己动手解引用。

> Go语言中的取址符是&，放到变量前使用，就会返回相应变量的内存地址。
>
> 一个指针变量，其作用就是只想一个值的内存地址。
>
> Go语言中，定义指针，形如
>
> var ip *int;
>
> **如何使用指针？**
>
> go语言中，通过在指针类型前加上*号，来获取指针的内容。
>
> **如何使用结构体指针？**
>
> 指向结构体的指针，称为结构体指针。
>
> 结构体指针，使用 "." 操作符来访问结构体成员，所以B对。
>
> 可以使用结构体变量名称的方式来访问，即*p,获取结构体的内容，所以A对。

A对，指针本身就是引用类型，可以通过“.”的方式调用其成员属性或方法。 然后看B和C，“\*”是根据指针地址去找地址指向的内存中存储的具体值，“&”是根据内存中存储的具体值去反查对应的内存地址。题目中已经说明了p是指针，也就是内存地址，要使用变量(这里是调用成员属性)，当然是要先根据内存地址获取存储的具体内容，选\*p。 D项，Go不支持这种调用写法。

## 序列化

1. 序列化是指吧对象（可以是stuct,string）按照协议编码填充头部和内容 变成二进制字节码 反射是指通过对象本身获取类型、方法、字段元信息以及对象的value ，深知事执行对象本身的调用 和系列化是两个事情，两个东西可以相互独立存在
2. 结构体在序列化时非导出变量（以小写字母开头的变量名）不会被encode，因此在decode时这些非导出变量的值为其类型的零值。
3. 序列化通常将类型结构传入标准库或第三方包，类型结构中没有大写的变量未导出，对第三方包不可见，无法进行任何操作，依旧是默认的零值。

4. 小写字母的变量还是存在的，只是不可显示，因为无法初始化，所以值为零值。
5. Golang支持反射，反射最常见的使用场景是做对象的序列化。
6. serialization，有时候也叫Marshal & Unmarshal。例如，Go语言标准库的encoding/json、encoding/xml、encoding/gob、encoding/binary等包就大量依赖于反射功能来实现。

## 包

1. 同级目录下，只能有一个包名，但包名可与该级目录名不一致。
2. import后面的最后一个元素应该是路径，就是目录，并非包名。

## 返回值

1. 如果失败原因只有一个，则返回bool
2. 如果失败原因超过一个，则返回error
3. 如果没有失败原因，则不返回bool或error
4. 如果重试几次可以避免失败，则不要立即返回bool或error

## 异常

1. 在程序开发阶段，坚持速错，让程序异常崩溃
2. 在程序部署后，应恢复异常避免程序终止
3. 对于不应该出现的分支，使用异常处理

## 内存泄漏

1. golang中检测内存泄露主要依靠的是pprof包
2. 内存泄露不可以在编译阶段发现

## 内存回收

下面代码中的指针p为野指针，因为返回的栈内存在函数结束时会被释放？

<img src="https://uploadfiles.nowcoder.com/images/20171205/4155837_1512460613518_F1ED8EA13FCD6212A852A8C466F78716" alt="img" style="zoom:150%;" />

错误。不会被释放。内存逃逸。

NewTimeMatcher函数返回一个地址，那么变量p就是一个结构体指针，这样在函数结束返回栈内存之后，这块内存仍然存在，需要GC来回收这块内存。

golang对局部变量的生命周期不做假设，而是根据是否被引用了来决定对象被创建在堆上还是栈上。这一过程称之为内存escape。

GO语言的内存回收机制规定，只要有一个指针指向引用一个变量，那么这个变量就不会被释放，因此在GO语言中返回函数参数或临时变量是安全的。

## 其他

1. 不可以var j MyInt = i.(MyInt)
2. 不可以使用var x = nil、var x string = nil
3. 无论是RWMutex还是Mutex，与Lock()对应的都是Unlock()
4. go用 ^ 或者 ! 进行取反
5. 错误是业务过程的一部分，而异常不是。

## GoStub

GoStub框架的使用场景很多，依次为：

1. 基本场景：为一个全局变量打桩
2. 基本场景：为一个函数打桩
3. 基本场景：为一个过程打桩
4. 复合场景：由任意相同或不同的基本场景组合而成
5. 不可以为结构体的成员方法打桩

## Go Mock

1. GoMock可以对interface打桩
2. GoMock打桩后的依赖注入可以通过GoStub完成
3. GoMock不可以对类的成员函数打桩，不可以对函数打桩

mock对象的注入：mock对象的行为都注入到控制器以后，我们接着要将mock对象注入给interface，使得mock对象在测试中生效。

在使用GoStub框架之前，很多人都使用土方法，比如Set。这种方法有一个缺陷：当测试用例执行完成后，并没有回滚interface到真实对象，有可能会影响其它测试用例的执行。所以建议大家使用GoStub框架完成mock对象的注入。

1. 全局变量可通过GoStub框架打桩
2. 过程可通过GoStub框架打桩
3. 函数可通过GoStub框架打桩
4. interface可通过GoMock框架打桩

## Go Vet

1. go vet是golang自带工具go tool vet的封装
2. go vet可以使用绝对路径、相对路径或相对GOPATH的路径指定待检测的包
3. go vet可以检测出死代码
4. 执行go vet database时，不可以对database所在目录下的所有子文件夹进行递归检测。go tool vet package1 package2 ； go tool vet 才可以递归。

## Go Convey

1. goconvey是一个支持golang的单元测试框架
2. goconvey能够自动监控文件修改并启动测试，并可以将测试结果实时输出到web界面
3. goconvey提供了丰富的断言简化测试用例的编写
4. goconvey可以与go test集成