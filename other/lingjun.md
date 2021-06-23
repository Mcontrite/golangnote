## Golang面试基础知识点大全

## **数组 (array)**

数组是golang中最基本的数据结构之一。数组的定义方式如下：

```go
var arr [n]type
```

其中n是数组的长度，type是数组中元素的数据类型。例如：

```go
var arr [3]int
```

定义数组长度可以省略，用"..."来代替，golang会自动根据元素的个数来计算长度。**例如下面定义了一个长度为3的数组，三个元素分别是整数1、2、3，

```go
arr := [...]int{1, 2, 3}
```

另外，一定要注意，**长度是数组类型的一部分**。所以[2]int和[3]int是不同的类型。所以下面的例子编译通不过，会提示“invalid operation: arr1 == arr2 (mismatched types [2]int and [3]int)”。

```go
func main() {  
    arr1 := [2]int{1, 2}  
    arr2 := [3]int{1, 2}
  	if arr1 == arr2 {    
        fmt.Println("equal")  
    }
}
```

最后要注意，**数组作为参数传递给函数时，是值传递；所以在函数内对数组做任何修改，不会影响到调用者**。例如，下面的例子最后输出是"[1, 2, 3]"。

```go
func test(arr [3]int) {  
    arr[0] = 10
}
func main() {  
    arr1 := [3]int{1, 2, 3}  
    test(arr1)  
    fmt.Println(arr1)
}
```

## **切片(slice)**

切片是golang中最常用的数据结构之一，是对数组的封装，可以动态的扩容。扩容机制很简单，当切片的长度小于1024时，直接双倍扩大，例如之前slice的cap（容量）是6，直接扩大到12。当切片的长度大于1024时，每次扩大1/4，直到满足需求为止；例如之前slice的cap为1024，首先扩大到1024 + 1024/4 = 1280，然后判断是否达到需求的大小，如果达到了，就结束了，没有达到，就继续扩大1280 + 1280/4 = 1600，以此类推，直到满足需求为止。

不过切片没有对应的缩减机制，也即是当容量很大，而实际长度很小时，不会自动缩小切片的容量。之所以没有提供缩减机制，我理解还是由于应用场景千奇百怪，golang不能替应用层做决定，否则可能出现不断的抖动。换个角度想一想，我们可以一开始就初始化一个容量很大，但实际长度很小的切片，如果golang会自动缩减，那么我们刚刚初始化的slice就可能被golang自动缩小。这种缩减机制应该是由应用层来决定，因为每个应用最清楚自己的应用场景。

既可以用直接赋初值的方式初始化slice，也可以用make创建slice。下面的例子就是直接初始化了一个slice，

```
s := []int{1, 2, 3}
```

很多人不知道的是，**其实可以为元素指定一个索引**。例如，下面的例子中，元素19的索引是5。第一个元素2没有指定索引，默认就是0；19之后的元素4也没有指定索引，默认就是下一个索引，也就是6。

```go
func main() {  
    s := []int{2, 5: 19, 4}  
    fmt.Printf("len(s)=%d, cap(s)=%d, s: %v\n", len(s), cap(s), s)
}
```

上面程序的输出结果为：

```go
len(s)=7, cap(s)=7, s: [2 0 0 0 0 19 4]
```

用make来创建slice的格式如下：

```go
make([]T, size, cap)
```

第三个参数可以省略。当不提供第三个参数时，创建的slice的长度和容量相同，例如，下面就创建了一个长度和容量均为3的slice，

```go
s := make([]int, 3)
```

注意：**这时往slice中append新元素是从第4个(index为3)位置开始添加，而不是从第1个(index为0)位置添加**，**前三个元素为零值**，

```
s = append(s, 4) // The slice contains [0, 0, 0, 4]
```

在上面的例子中，由于slice初始的长度和容量均为3，所以向slice中添加新的元素时，slice会自动扩容到3*2=6。注意，**这时会为slice分配了新的数组存储空间。如果在扩容之前有某个指针指向了slice的某个元素，那么扩容之后，无论如何修改slice的内容，之前那个指针永远指向扩容之前的旧值**。这里就不举例了。

**可以将一个切片追加到另一个切片后面，这时需要使用 "..."操作符；或者直接列出所有元素。**例如下面两种方式都是正确的，

```go
//method 1
s1 = append(s1, s2...)
//method 2
s1 = append(s1, 4, 5, 6)
```

另外，也可以基于一个slice做切片操作，例如，下面的例子中，s2就是基于s1产生的一个新切片。注意，**s2和s1底层实际上是指向相同的数组，如果通过s2修改的某个元素的值，通过s1可以看到修改后的值**。

```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[2: 4]
```

新切片s2的容量为cap(s1) - i。上面的例子中，i为2，cap(s1)为5，所以cap(s2)等于3。值得一提的是，**其实还可以提供第三个参数，用来限制新切片的容量cap；如果提供第三个参数，其值必须大于前两个参数**。例如，下面的例子中，因为提供了第三个参数7，所以s2的容量为7-3=4。

```go
s1 := []int{1, 2, 3, 4, 5, 6, 7, 8}
s2 := s1[3:5:7]
```

基于一个slice做切片操作时，两个参数都是可选的。当第2个参数省略时，默认值就是原slice或array的长度。例如下面的例子中，s1[2:]实际上等价与s1[2: 5]，所以s2的值为[3, 4, 5]。

```go
s1 := []int{1, 2, 3, 4, 5}s2 := s1[2: ]
```

这里有一个坑，**当省略第二个参数时，第一个参数的值一定不能大于原slice的长度，否则会panic**。例如，下面的例子中，s1的长度和容量分别是2和8。当基于s1做切片操作时，第一个参数的值3大于s1的长度2，所以会panic：“panic: runtime error: slice bounds out of range [3:2]”。

```go
s1 := make([]int, 2, 8)s2 := s1[3:]
```

最后，捎带提一句，**slice是不能用==来比较的，也就不能用做map的key**。

## **map**

map也是golang中最常用的数据结构之一。map是一堆无序的key:value对的集合，所以遍历map时是无序的。注意，map中的key必须是可比较类型。

map使用之前，一定要初始化。既可以通过直接赋初值的方式初始化，也可以通过内置函数make初始化。如果只是声明一个map，而没有初始化，那么该map就等于nil。例如，下面就是一个nil map，

```go
var m map[string]int
```

向nil map中写入数据肯定是不行的，会引起panic。**但是却可以从nil map中读取数据，返回零值。**

**那么如何判断是由于nil map返回零值，还是map中本来就包含这么一个零值的键值对？答案就是通过第二个bool类型的返回值\**，如果为true，表示是从map中正常读取到的值，否则就是异常情况（map为nil，或者map中不存在对应的key）**。例如：

```go
var m map[string]int
if v, ok := m["hello"]; ok {    
    fmt.Println(v)
} else {    
    fmt.Println("m is nil or m doesn't have key 'hello'")
}
```

map的基础数据结构定义在src/runtime/map.go里的hmap。不难看出，真正的数据都是存储在由指针成员指向的内存空间。所以，**当map作为参数传递给函数后，在函数内对map做增、删、改操作，都会影响到调用者**。例如，在函数change内对map的修改，在main函数内都能看到，最后输出结果为：map[g2:5 h1:3 h2:4]。

```go
func change(m map[string]int) {  
    m["h1"] = 3  
    m["h2"] = 4  
    delete(m, "g1")  
    m["g2"] = 5
}
func main() {  
    m := make(map[string]int)  
    m["g1"] = 1  
    m["g2"] = 2  
    change(m)  
    fmt.Println(m)
}
```

最后值得一提的是，**如果map中value不可寻址，无法直接通过 "m[key].val = newVal" 的方式直接修改，否则编译无法通过**。例如，下面的例子中是错误的

```go
type student struct {  
    age int
}
func main() {  
    m := make(map[int]student)  
    m[1] = student{23}  
    m[1].age = 24 // invalid  
    fmt.Println(m[1].age)
}
```

解决方案就是将map的value改成*student。修改后的代码如下：

```go
type student struct {  
    age int
}
func main() {  
    m := make(map[int]*student)  
    m[1] = &student{23}  
    m[1].age = 24  
    fmt.Println(m[1].age)
}
```

## **channel**

channel是golang的一种特殊的数据结构，主要用于goroutine之间通信，从而避免共享内存的方式来通信。

channel通过内置函数make来创建，创建的时候可以指定缓冲的大小，也可以不指定。注意，**有缓冲的channel是异步操作；而没有缓冲的channel是同步操作**。

如果只是声明一个channel，而没有使用make来初始化，那么该channel就等于nil。例如，下面声明的channel就是一个nil channel

```go
var ch chan intgo
```

注意，**不管是向一个nil channel发送数据，还是从nil channel接收数据，都会永久阻塞**。所以使用channel之前，一定要用make初始化。

channel使用完之后，使用内置函数close关闭。注意，**不能重复关闭一个channel，否则会panic**。从src/runtime/chan.go中的函数closechan可以看到具体的实现，

```go
if c.closed != 0 {    
    unlock(&c.lock)    
    panic(plainError("close of closed channel"))
}
```

**向一个已经关闭的channel发送数据，同样会引起panic**。从src/runtime/chan.go中的函数chansend也可以看到具体的实现，

```go
if c.closed != 0 {    
    unlock(&c.lock)    
    panic(plainError("send on closed channel"))
}
```

但是可以从一个已经关闭的channel中接收数据，直到channel没有数据为止。注意，当一个channel已经关闭，并且里面也没有数据时，如果尝试从channel里面读取数据，会读取到零值(例如：对于int就是0，对于string就是空字符串"")。

**那么如何判断是由于channel关闭了返回零值，还是channel里面的数据本来就是零值？答案就是通过第二个bool类型的返回值，如果为true，则表示第一个返回值是从channel里面正常读取到的数据，否则表示channel已经关闭了**。例如，下面的例子就是通过第二个返回值来判断v是否是从channel中正常读取到的数据，

```go
for {    
    if v, ok := <-ch; ok {      
        fmt.Printf("Received: %v\n", v)    
    } else {      
        break    
    }  
}
```

最后要注意一点，**不能关闭单向只读channel**，否则编译会报错。例如，下面例子是错误的，

```go
func test(ch <-chan int) {  
    for v := range ch {    
        fmt.Println(v)  
    }  
    close(ch)
}
```

## **string**

**golang中的string是不可改变的，这一点与Java类似**。下面的代码尝试直接修改string是错误的，编译无法通过

```go
func main() {  
    str := "hello"  
    str[0] = 'g' //invalid
}
```

**string的底层其实是byte数组，len(str)返回的是字节数**。例如，下面的例子最后输出12，因为两个汉字都分别占用3个字节。

```go
func main() {  
    str := "hello,世界"  
    fmt.Println(len(str))
}
```

**但是用range遍历string时，是按照实际的字符来遍历的**。例如，下面例子中，v的类型是rune，其实就是int32的别名。

```go
func main() {  
    str := "hello,世界"  
    for i, v := range str {    
        fmt.Printf("%d, %c\n", i, v)  
    }
}
```

上面的例子输出：

```
0, h
1, e
2, l
3, l
4, o
5, ,
6, 世
9, 界
```

在golang中，**string是以utf8编码的unicode文本。注意: unicode是一种字符集，而utf8是对unicode的一种编码规则**。utf8解决了string的存储和传输的问题。

golang提供了一个包strings，里面有很多实用的函数可以用来处理string。这其中容易混淆的是下面两组函数，

```go
func TrimLeft(s string, cutset string) string
func TrimRight(s string, cutset string) string
func TrimPrefix(s, prefix string) string
func TrimSuffix(s, suffix string) string
```

**TrimPrefix是删除s头部的prefix字符串，如果s不是以prefix开始，则直接返回s。而TrimLeft则是删除s头部连续的包含在cutset中的字符**。例如，下面的例子输出：“abab,world”，

```go
func main() {  
    str := "ababab,world"  
    fmt.Println(strings.TrimPrefix(str, "ab"))
}
```

如果将上面的例子第10行改成strings.TrimLeft，则最后输出",world"。

**同理，TrimSuffix是删除s尾部的suffix字符串，如果s不是以suffix结尾，则直接返回s。而TrimRight则是删除s尾部连续的包含在cutset中的字符**。这里就不举例了。

## **关于分号(Semicolons)**

golang的正式语法是以分号作为每一条语句的终止符，但是一般情况下，程序员不需要自己输入分号，因为golang会自动插入分号。golang spec对此有明确的说明，如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/ibpInLhR6azQdbyMDRKM71LeUM82yw3xBkFKOMxuHrDC0lkF1icrUtFI2DZ7mMzzSjYXxu4CRmshFDfDKZByyt5g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这就很好的解释了为什么在初始化一个结构体，为每个字段赋值时，在最后一个字段后面还要加一个逗号。例如，**下面的例子中，如果没在"ben"后加一个逗号，那么golang会自动在"ben"后插入一个分号，从而将一个完整的语句强行拆分开了，编译就会报错**。

```go
s := student{    
    age:  25,    
    name: "ben",
}
```

当然，上面的例子中，还可以将最后的"}"直接写在"ben"后面，而不是另起一行，如下。只是这么写不太美观而已。

```go
s := student{    
age:  25,    
name: "ben"}
```

很多从其它语言转到golang的程序员，都很好奇为什么"{"必须要放在一条语句的结尾，而不能另起一行，否则编译就会报错。例如，**下面的语法是错误的，**因为golang会自动在"i++"后面插入一个分号，把原本是一个完整的逻辑单元强行拆分开了。

```go
for i := 0; i < 10; i++ 
{    
	fmt.Println(i)
}
```

注意，不是所有的"{"都必须放在一行的结尾。例如下面switch的例子，"{"在另起一行的开头，语法也是正确的。**根据golang spec的说明，golang只会在break、continue、fallthrough、return这四个关键字后插入分号，显然switch不在其中，所以golang不会在switch后自动插入分号，所以下面这种写法语法正确**。

```go
switch
{  
case i < 10:    
	fmt.Println("i < 10")  
case i < 20:    
	fmt.Println("i < 20")  
default:    
	fmt.Println("i >= 20")
}
```

当然，为了风格的一贯性，还是建议将“{"写在前一行的结尾处，如下：

```go
switch {  
    case i < 10:    
    fmt.Println("i < 10")  
    case i < 20:   
    fmt.Println("i<20")  
    default:    
    fmt.Println("i >= 20")
}
```

golang spec中关于分号的最后一条规则，估计很多人看不明白。这里举一个例子说明。下面的例子中，**一条复杂的语句完全放在了一行，{}里面的两条语句之间必须加一个逗号，但最后一条语句后面的逗号可以省略**。

```go
for i := 0; i < 3; i++ {a:=i+1; fmt.Println(a)}
```

当然，最后加上一个逗号，语法也是正确的，

```go
for i := 0; i < 3; i++ {a:=i+1; fmt.Println(a);}
```

## **变量定义**

golang的变量，既可以通过var定义，也可以直接通过":="直接赋初值的方式定义，例如下面几种方式都是合法的：

```go
var a int; a = 1
var b int = 2
var c = 3
d := 4
```


不管是上面哪种方式定义变量，要注意一点，变量定义之后，一定要使用，否则编译会出错。**golang不允许声明未使用的变量**。但是要注意，对于常量，则**允许定义未使用的常量**，例如，下面定义了一个常量PI，但程序中没有任何地方使用它，也是合法的。

```go
const PI = 3.14
```

对于用":="这种简短模式定义变量，一定要注意几点：1、**:=只能用在函数内部，也就是说不能用来定义全局变量**；2、不用提供数据类型，编译器会自动推导，如果要提供数据类型，那就要使用"="，而不是":="。

定义变量而不提供数据类型时，golang会自动推导类型，例如，下面的例子中，golang自动推导i, j的数据类型分别为int、string。

```
i := 4
var j = "hello"
```

但是要注意，**在不提供数据类型时，"="右边不能是nil，因为golang无法推导变量的类型**。例如下面的定义就是非法的，

```go
i := nil
var j = nil
```

如果要给一个变量赋nil，那必须要指定类型，下面几种写法都正确：

```go
type Interface interface {    
    Do()
}
a := Interface(nil)
var b = Interface(nil)
var c Interface = nil
var d Interface; d = nil
```

要注意，**nil只能赋给指针、channel、函数变量(func)、interface、map、slice这几种类型的变量**。所以诸如var s string = nil这种赋值语句是错误的。

在golang中，可以基于一个现有的数据类型定义一个新的类型，例如，下面的例子定义了一个新的数据类型MyInt，其底层的类型是int。

```go
type MyInt int
```

在上面的例子中，**虽然MyInt的底层类型是int，但是MyInt和int是两种不同的数据类型，因为golang是强类型的语言，所以MyInt类型的变量和int类型的变量之间不能相互赋值**。例如，下面的例子中，第4行是非法的，编译通不过。

```go
type MyInt int
var a MyInt
var b int = 2
a = b
```

但是并不意味着，所有新定义的类型和原类型的变量之间不能相互赋值，例如，下面的程序就是合法的，

```go
type MySlice []int
var ms MySlicevar 
s []int = []int{1, 2, 3}
ms = s
```

其实golang spec对于两个不同的变量之间在什么情况下可以相互赋值，有明确的说明，其中关键是：

**两个变量必须具有完全相同的底层类型，并且至少有一个不是defined type**。defined type有两种：

1. golang的内置类型，例如int, string, bool, float64等；
2. 使用关键字type定义的类型。

回头再来看上面的例子，**因为MySlice与[]int具有完全相同的底层类型，而且[]int不是defined type，所以相互赋值是合法的**。

这里要区分开类型定义(defined type)和别名(alias)两个概念。**为一个类型定义一个别名(alias)需要使用=。alias类型的变量和原类型的变量之间可以相互赋值**。例如，下面的例子中MyInt只是int的一个别名，所以a, b之间可以相互赋值。

```go
type MyInt = int  // MyInt is a alias of int
var a MyInt
var b int = 3
a = b
```

每个变量都有自己的作用域，这里要注意，**定义在不同作用域的两个同名变量，互不影响**。例如，下面例子中变量a定义了两次，但在不同的作用域，所以互不影响，在if里面的a的值为5，而if外的a的值为3。

```go
func main() {  
    a := 3  if a > 0 {   
        a, b := 5, 6   
        fmt.Println(a, b) 
    } 
    fmt.Println(a)
}
```

如果把上面的例子稍微修改一下，估计很多人就开始迷惑了。下面这个例子，将a+1的结果赋给b，但一定要注意，**这里是用if语句之外的a参与运算，因为此时if内的变量a尚未初始化完成，\**所以b的值是3+1=4\****。

```go
func main() { 
    a := 3 
    if a > 0 { 
        a, b := 5, a+1  
        fmt.Println(a, b) 
    }  
    fmt.Println(a)
}
```

上面的例子中出现了简单的多重赋值的场景（第8行）。下面这个例子中的多重赋值则容易让人迷惑。你先猜猜，最后结果输出什么？


```go
func main() {
    s := []int{1, 2, 3, 4, 5}
    i := 1  
    i, s[i+2] = 2, 10 
    fmt.Println(i, s)
}
```

对于这种多重赋值情况下的计算顺序，golang spec有明确的说明：

> The assignment proceeds in two phases. First, the operands of index expressions and pointer indirections on the left and the expressions on the right are all evaluated in the usual order. Second, the assignments are carried out in left-to-right order.

简单翻译一下，**分三步：**

**1、首先是计算等号左侧的"索引表达式"与"指针取址"；**

**2、计算等号右边的所有表达式的值；**

**3、赋值，也即是将等号右边的值按“从左到右”的顺序赋给左边的变量；**

回到前面的例子，首先计算索引表达式"i+2"，结果为3；然后计算右边的表达式，因为右边都是常数，没什么计算的；最后将2赋给i，将10赋给s[3]。所以最后输出结果为：

> 2 [1 2 3 10 5]

与其它语言类似，golang中对变量也有++和--操作，例如，下面的例子中i自增1，而j自减1。

```go
i++
j--
```

一定要记住，**++和--只能放在变量的后面，而不能在变量前面**。所以下面的写法是错误的，

```go
++i
--j
```

另外，**++和--形成的是语句(statement)，而不是表达式(expression)**。所以下面的写法是错误的，编译通不过，


```go
s[i++] = 5
s[j--] = 2
```

与C/C++一样，golang中也支持指针。可以将一个变量的地址赋给一个指针，例如：

```go
i := 1
p := &i
*p = 4
```

但是**指针不能指向一个常量的地址**，例如下面的写法就是错误的，编译通不过，


```go
const A = 4  
q := &A
```

对整型变量进行运算时要注意溢出问题。这里分两种情况，分别为无符号整数计算和有符号整数计算。先来看一个无符号整数计算的例子，

```go
func main() {  
    var a uint8 = 100 
    fmt.Println(a * a)
}
```

上面的例子中，因为a是uint8，只有一个字节(8 bit)，a*a的结果为10000，转换成二进制就是：

> 10 0111 0001 0000

因为**结果超过了8位，所以直接对结果进行截取，取最低的8个bit，所以结果为10000，也就是16**。

再来看一个有符号整数计算的例子，

```go
func main() { 
    var a int8 = 127
    fmt.Println(a + 1)
}
```

上面的例子中，a+1的结果为128，转换成二进制就是：

> 1000 0000

**因为a是有符号整数，而且只有8个bit，计算结果最高位为1，表示负数，所以最后输出为-128**。

最后再看一个有趣的计算，最后输出是什么？

```go
func main() { 
    a := 1 
    a += 010 
    fmt.Println(a)
}
```

这里要注意，**计算结果不是11，而是9，因为010是八进制，换成十进制就是8，所以结果是1+8=9**。 对于二进制、八进制、十六进制数的前缀表示，请参考：[golang 1.13新特性解读](http://mp.weixin.qq.com/s?__biz=MzU2MjM1NTY3Mw==&mid=2247484002&idx=1&sn=ca1c26085ed930a4566058f0c50a4adc&chksm=fc6b8ed4cb1c07c2d283a7c2e4133a15e64f88c6884b2f151e59e16688f913e6e7a478128df7&scene=21#wechat_redirect)。

## **关键字和预定义标识符**

golang中一共有25个关键字(keyword)，分别如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/ibpInLhR6azQdbyMDRKM71LeUM82yw3xBUiadHNTiazU1iaSZ04lQGQcX6BC4Gn5a82625XzGOQzxQnVLsdtxXar8Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

golang中预定义的标识符为：

![img](https://mmbiz.qpic.cn/mmbiz_png/ibpInLhR6azQdbyMDRKM71LeUM82yw3xBZubhbrzx9MCO7mZeiccOejWeNEu5CFHibcDyobT3njxS65H4FSU2e2Kw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里只记住一点，**关键字不能用来定义标识符(包括变量和类型)，而预定义的标识符则可以（当然不建议这么干）**。例如，下面的例子语法是正确的，因为make和nil都只是预定义标识符，可以用来定义类型和变量，

```go
type make int
func main() {  
    var a make = 9 
    nil := 8 
    fmt.Println(nil, a)
}
```

而下面的例子则是非法的，因为func和break都是关键字，不能用来定义类型和变量，

```go
type func int
func main() {  
    var a func = 9  
    break := 8  
    fmt.Println(break, a)
}
```

## **内置函数**

golang提供了下面这些内置函数，它们都属于预定义的标识符（见上一节"预定义标识符"），

> append len cap close copy delete complex real imag make new panic recover print println

**由于这些内置函数都没有标准的类型，所以他们不能赋给其它的函数变量**。例如，下面的例子就是非法的，因为不能将内置函数make赋给变量f，


```go
func main() { 
    f := make  
    s := f([]int, 3)  
    fmt.Println(s)
}
```

正确的写法应该是：


```go
func main() { 
    s := make([]int, 3)
    fmt.Println(s)
}
```

对于这些内置函数，每一个都有一些值得注意的点。例如**len的参数可以是string、array、slice、map、channel；而cap的参数则只能是array、slice、channel**。所以，下面的例子是非法的，因为cap的参数不能是map类型，

```go
func main() { 
    m := map[string]int{}
    fmt.Println(cap(m))  //invalid
}
```

再例如，copy函数既可以用来拷贝slice，也可以将一个string拷贝到[]byte。调用格式如下：


```go
copy(dst, src []T) int
copy(dst []byte, src string) int
```

这里要注意，**拷贝的长度是len(src)和len(dst)的最小值**，例如，下面的例子中，s1和s2的长度分别为3和2，所以只会拷贝2个元素，返回值是实际拷贝的元素数量，也就是2。

```go
func main() { 
    s1 := []int{1, 2, 3}
    s2 := []int{8, 9} 
    l := copy(s1, s2)
    fmt.Println(l, s1)
}
```

我曾经见过一个有意思的面试题，要求用一行代码来比较两个正整数的大小，返回其中较小的值。这个题就可以利用copy巧妙的实现，示例如下：

```go
func min(a, b int) int { 
    return copy(make([]int, a), make([]int, b))
}
func main() {  
    fmt.Println(min(5, 9))
}
```

关于其它比较重要的内置函数，例如panic/recover，new和make，在后续的文章中会单独重点阐述。

## **类型转换**

与其它语言(比如Java, C++)类似，golang也支持类型转换。但是与其它语言不同的是，**类型转换时，要用小括号将需要转换的变量括起来，而不是将要转换到的目标类型括起来**。例如，下面的例子就是将int8类型转换成int64，

```go
var i int8 = 8
var j int64j = int64(i)
```

下面的写法是错误的，因为没有将要转换的变量i括起来，

```go
j = (int64)i  // invalid
```

有的类型转换就复杂一些，很容易犯错。例如，下面例子中的两条语句的含义完全不同。第一条是将p转换成类型*Point，而第二条相当于*(Point(p))，先将p转换成Point，然后再解引用。

```go
(*Point)(p)
*Point(p)
```

而下面两个转换语句的含义则完全一样，都是将x转换成func() int，

```go
(func() int)(x) 
func() int(x)
```

golang中的类型转换的条件和规则其实比较复杂，限于篇幅，这里就不展开阐述。但是记住一点，**当你不确定的时候，将目标类型和需要转换的变量都括起来，避免歧义**。

提到类型转换，就不得不提type assertions。例如，下面就是一个type assertion的例子，

```go
var i interface{} = 8 
j, ok := i.(int) 
fmt.Println(j, ok)
```

type assertions的格式为：

```go
x.(T)
```

这里要注意，**x必须为接口类型，可以是interface{}，或者是自定义的一个接口类型**。T既可以是接口类型，也可以不是接口类型。**当T不是接口类型时，T必须与x的动态类型完全相同。在这种情况下，T肯定也实现了x的静态接口类型**。例如，下面例子的第19行就用到了type assertion，其中的student就是T，不是一个接口类型，所以它必须与参数h的动态类型完全一样，而且student也实现了x的静态接口类型Interface1。

```go
type student struct {  
    age  int 
    name string
}
type Interface1 interface {
    Do1()
}
func (s student) Do1() { 
    fmt.Println("student Do1()")
}
func test(h Interface1) { 
    v, ok := h.(student) 
    v.Do1() 
    fmt.Println(v, ok)
}
func main() {  
    var s Interface1 = student{25, "ben"} 
    test(s)
}
```

**当T是接口类型时，x.(T)判断x的动态类型是否实现了接口类型T，如果是，则第二个返回值为true，否则为false**。将上面的例子稍微修改一下，student同时实现了Interface1和Interface2。在第27行，T是接口类型Interface2，虽然h的静态类型是Interface1，但其动态类型student实现了Interface2，所以type assertion返回了true(第二个返回值)。

```go
type student struct {  
    age  int 
    name string
}
type Interface1 interface { 
    Do1()
}
type Interface2 interface { 
    Do2()
}
func (s student) Do1() {  
    fmt.Println("student Do1()")
}
func (s student) Do2() {  
    fmt.Println("student Do2()")
}
func test(h Interface1) { 
    v, ok := h.(Interface2)
    v.Do2()  
    fmt.Println(v, ok)
}
func main() { 
    var s Interface1 = student{25, "ben"} 
    test(s)
}
```

## **按位取反**

在C/C++/Java中，位取反操作符是  **\~**，**但是在golang中，位取反操作符则是^**。在golang中没有~这个操作符。切记！注意，^用作一元操作符时表示位取反，用作二元操作符时表示异或。

**对于无符号数的取反，很简单，每一个bit直接取反即可，也即是0变1，1变成0即可**。例如，下面的例子中对类型为uint8的2取反，因为2的二进制是0000 0010，取反之后就是1111 1101，也就是253，所以最后输出253，

```go
func main() {  
    var i uint8 = 2
    fmt.Println(^i)
}
```

对于有符号数的取反，则稍微绕一点。**如果x是一个有符号数，那么\^x其实就等于x与-1异或的结果，也即是-1^x**。把上面的例子稍微修改一下，将2的类型改成int8，2的二进制是0000 0010，而-1的二进制是1000 0001，两者异或，结果就是1000 0011，也就是-3，所以最后输出-3。

```go
func main() { 
    var i int8 = 2 
    fmt.Println(^i)
}
```

**对于有符号整数的取反，有一个快速的计算公式，就是-(x+1)**。例如2取反就是-3；-5取反就是4。

## **iota**

iota是golang的一个预定义的标识符，用于定义常量，而且**只能用于定义常量**。**iota自身表示从0开始的连续的常量**。例如，下面的例子用iota定义了四个常量，


```go
const (  
    _  = iota             // (iota = 0, unused)  
    KB = 1 << (10 * iota) // KB == 1<<(10*1) (iota == 1)  
    MB                    // MB == 1<<(10*2) (iota == 2)  
    GB                    // GB == 1<<(10*3) (iota == 3)
)
```

上面的例子比较简单，虽然第一个iota没有使用，但是下一行的iota也加1了。下面这个例子就具有一定的迷惑性。这里必须要明确一点，**只要遇到const，iota的值就是0，随后无论怎么定义常量，也无论是否使用iota，iota的值都是逐行加1**。所以最后计算常量E时，iota的值为6，E的值也就是6。另外，值得注意的是，**因为常量B和D都没有对应的表达式，所以沿用前一个常量的表达式；所以B沿用A的表达式，值为iota+10 = 11，D沿用C的值，还是100**。

```go
const (
    A = iota + 10 // A == 10 (iota == 0)  
    B             // B == 11 (iota == 1) 
    _             // (iota == 2, unused)
    _             // (iota == 3, unused)
    C = 100       // (iota == 4, unused)
    D             // (iota == 5, unused) 
    E = iota      // E == 6 (iota == 6)
)
```

要强调一点，**只要遇到const，iota的值就会从0开始**。所以，下面的例子中，常量A、B、C的值都是0，而D的值则是1。

```go
const A = iota
const B = iota
const ( 
    C = iota  
    D
)
```

还**可以将多个常量定义在同一行，这时在同一行的多个iota的值都是相同的**。例如下面的例子中，每一行定义了两个常量，

```go
const (  
    A1, A2 = 10 * iota, 10*iota + 1 //A1==0, A2==1  
    B1, B2                          //B1==10, B2==11 
    C1, C2                          //C1==20, C2==21
)
```

在实际的开发中，**itoa的一个常见的用途就是模拟定义枚举类型**，例如，


```go
type DIRECTION int
const (  
    EAST DIRECTION = iota 
    SOUTH 
    WEST 
    NORTH
)
```

最后值得一提的是，**iota很容易错误的拼写成itoa**，因为C/C++的标准库有itoa这个函数：）。这里分享一个小窍门，**可以把iota看成是"int of the abc"的首字母缩写，**这样就不容易写错了。

## **比较操作符**

golang中有下列比较操作符，


```go
==    equal
!=    not equal
<     less
<=    less or equal
>     greater
>=    greater or equal
```

**其中==和!=适用于可比较(comparable)的操作数，而其它四个操作符适用于可排序(ordered)的操作数**。哪些类型的操作数可比较、可排序呢？见下表，

|               | 可比较? | 可排序? |
| ------------- | ------- | ------- |
| 布尔值        | Y       |         |
| 整数          | Y       | Y       |
| 浮点数        | Y       | Y       |
| 复数          | Y       |         |
| 字符串        | Y       | Y       |
| 指针          | Y       |         |
| 通道(channel) | Y       |         |
| 接口          | Y       |         |
| 结构体        | Y       |         |
| 数组          | Y       |         |

表格中没有出现的数据类型都是既不可比较，也不可排序**。**例如**slice、map和函数都是不可比较的**。下面的例子中，比较两个slice值，是非法的，编译通不过，

```go
func main() {  
    s1 := []int{1, 2} 
    s2 := []int{1, 2}  
    if s1 == s2 {  // invalid   
        fmt.Println("equal")  
    }
}
```

但是要注意一点，**虽然slice、map和函数不可比较，但它们可以和nil进行比较**。例如，下面的例子是合法的，

```go
func main() { 
    s1 := []int{1, 2} 
    s2 := []int{1, 2}  
    if s1 != nil { 
        fmt.Println("s1 is not nil") 
    }
}
```

对于结构体，**只有当结构体的所有字段都可比较时，结构体值才是可比较的。如果两个结构体值的所有非空字段都相等，那么它们就是相等的**，否则不相等。数组也是一样，**所有元素可比较，数组就可比较；所有元素都相等，两个数组就相等**。

**channel虽然可以相互比较，但只有同一个make函数创建出来的channel才相等**。例如，下面的例子中，函数test判断两个channel是否相等。只有当两个channel是同一个make函数创建的，才会返回true。说白了，当两个channel实际上是同一个channel时才返回true。所以最后输出true和false。

```go
func test(ch1, ch2 chan int) bool {  
    return ch1 == ch2
}
func main() {  
    ch1 := make(chan int)  
    ch2 := make(chan int) 
    fmt.Println(test(ch1, ch1)) 
    fmt.Println(test(ch1, ch2))
}
```

**接口值相互比较时，只有它们具有完全相同的动态类型，并且具有完全相同的动态值，才认为是相等的**。这里有一个很多人都会踩的坑，就是**判断接口值是否为nil时，只有当接口值的动态类型和动态值都是nil时，才与nil相等**。例如，下面的例子中接口值i的动态类型和动态值都是nil，所以i等于nil；虽然mi的值是nil，但其动态类型是*MyInt，不为nil，所以mi不等于nil。所以最后输出true和false。

```go
type Interface interface { 
    Do()
}
type MyInt int
func (mi MyInt) Do() { 
    fmt.Println("MyInt Do()")
}
func main() {
    var i Interface 
    var mi Interface = (*MyInt)(nil)  
    fmt.Println(i == nil)  
    fmt.Println(mi == nil)
}
```

## **range**

range用在for循环中，可以用来遍历array、slice、string、map和channel。**当遍历array时，range后面的表达式，可以是指向数组的指针，注意：只有遍历数组才能用指针**。例如，下面的例子中pa是一个指针，

```go
func main() { 
    a := [3]int{1, 2, 3} 
    pa := &a  
    for i, v := range pa {   
        fmt.Println(i, v)  
    }
}
```

要注意，**range后面的表达式在遍历开始前就已经计算完成了，换句话说，range是在原来数据结构的副本上遍历**。例如，下面的例子中，虽然遍历过程中不断往slice中插入新的元素，但遍历次数始终是3次；但是要注意，**slice的副本与原slice开始时是指向同一个底层的array**，

```go
func main() {  
    a := []int{1, 2, 3} 
    for i, v := range a {
        a = append(a, v)  
        fmt.Println(i, v, a)  
    }
}
```

上面的例子最后输出为：

```go
0 1 [1 2 3 1]
1 2 [1 2 3 1 2]
2 3 [1 2 3 1 2 3]
```

**用range遍历map的顺序是不确定的**。如果既想使用map，又希望有确定的遍历顺序，那可以使用linkedmap，请参考下面的链接，

> https://github.com/ahrtr/gocontainer/blob/master/map/linkedmap/linkedmap.go

**如果在遍历map的过程中，删除了某条记录(key/value)，如果该记录还未遍历到，那之后肯定不会遍历到了。如果遍历map的过程中，新增了一条记录，那么在随后的遍历中，该条记录可能会遍历到，也可能遍历不到，也就是不确定**。例如，下面例子在遍历map的过程中，不断新增新记录。如果你尝试运行这段代码，会发现每次的运行结果可能不同。

```go
func main() {  
    m := map[int]int{    
        1: 1,  
        2: 2,  
        3: 3,  
    }
  	for k, v := range m {  
        m[k+10] = v + 10 
        fmt.Println(k, v, m)  
    }
}
```

为什么在遍历map的过程中增加新的记录会导致遍历结果不确定？对这个问题，golang spec没有明确说明。**零君个人的理解还是和map的实现原理相关，也就是和map的底层数据结构相关**。map实际上是一个hashTable，由一组bucket构成，而每个bucket最多包含8条记录。首先，range是值拷贝这点，对map应该还是适用的；但是注意，同样也只是拷贝了最上层的结构体，并没有拷贝底层的bucket数据结构。其次，当遍历过程中向map中插入新记录时，如果新元素只是添加到现有的某个bucket中，如果该bucket还未遍历到，那随后的遍历中，肯定能遍历到该新记录；如果由于插入新元素导致了bucket数据结构动态扩容，那么会为map分配新的存储空间，而range中map的副本依然指向老地址，所以这种情况下，随后的遍历肯定就遍历不到新记录。**注意，这只是零君个人的理解，仅供参考**！

## **defer**

defer的作用在于延迟执行某个函数调用，直到当前函数return，或者发生了panic。**如果一个函数中有多个defer语句，是按照后进先出的顺序执行**。例如，下面例子的输出结果为：

> exiting... 
>
> hello2 
>
> hello1

```go
func test(s string) {  
    fmt.Println(s)
}
func main() {  
    defer test("hello1") 
    defer test("hello2") 
    fmt.Println("exiting...")
}
```

**虽然defer后面的函数调用会延迟执行，但函数的参数会即时计算**。例如，下面的例子中，虽然defer语句之后，变量v的值发生了改变，但最后还是输出1。

```go
func test(v int) { 
    fmt.Println(v)
}
func main() {
    v := 1  
    defer test(v)  
    v += 2  
    fmt.Println("exiting...")
}
```

另外，**defer后面如果是方法调用，那么receiver的值也是即时计算**。例如，下面的例子的第15行，首先会即时计算pv.test()，得到receiver的值，而后面的方法test()要等main函数返回时才执行。所以最后的输出结果为：

> 3 
>
> exiting... 
>
> 4

```go
type MyInt int
func (i *MyInt) test() *MyInt {  
    *i += 1
    fmt.Println(*i) 
    return i
}
func main() { 
    var v MyInt = 2 
    pv := &v  
    defer pv.test().test() 
    fmt.Println("exiting...")
}
```

## **panic & recover**

panic和recover是golang的两个内置函数，主要是处理运行时的异常或程序定义的错误。首先要注意的就是，**panic之后的语句都是执行不到的**。例如，下面的例子中，最后两行是执行不到的。执行结果为：

> start 
>
> runtime panic: 10


```go
func main() {  
    defer func() {  
        if err := recover(); err != nil {   
            fmt.Printf("runtime panic: %v\n", err)   
        }  
    }()
 	fmt.Println("start") 
    panic(10) 
    panic(2)  
    fmt.Println("end")
}
```

其次，要注意的是，**recover只有在defer函数中直接调用才有效**。例如，**下面写法是错误的，因为recover没有放在defer函数中执行**。程序最后由于panic异常退出。

```go
func main() { 
    if err := recover(); err != nil {  
        fmt.Printf("runtime panic: %v\n", err)  } 
    panic(10)
}
```

下面的例子也是错误的，**虽然recover在defer函数中，但并不是直接调用，而是又嵌套了一层defer**。同样，程序最后由于panic异常退出

```go
func main() { 
    defer func() {   
        defer func() {    
            if err := recover(); err != nil {   
                fmt.Printf("runtime panic: %v\n", err)    
            }   
        }() 
    }()  
    panic(10)
}
```

在上面两种错误的写法中，recover函数都是返回nil，并没有捕获到panic的异常。

## **闭包(closure)**

闭包通常是与first class相关联的一个概念。**将一个匿名函数作为返回值直接返回给调用者后，该匿名函数还可以继续访问原来函数中的变量，这就是闭包**。例如，下面就是一个闭包的例子，函数test的返回值是一个函数，在main函数中调用这个匿名函数时，依然可以访问原来函数test中的变量i。最后的输出为“9 10”。

```go
func test() func() {  
    i := 8  
    return func() {    
        i++    
        fmt.Printf("%d  ", i)  
    }
}
func main() {  
    f := test() 
    f()  
    f()
}
```

在golang中，**闭包还有另一种用法，就是直接创建一个goroutine来运行匿名函数**。下面的例子就体现了这种用法。

```go
func test() {  
    i := 8  
    go func() {   
        i++    
        fmt.Printf("%d  ", i) 
    }()
}
func main() {  
    test() 
    time.Sleep(time.Second)
}
```

使用闭包的时候，有一个很容易踩的坑。例如，下面的例子，**本来期望输出"1 2 3"，结果发现实际输出是"3 3 3"，因为几个goroutine执行匿名函数时访问的实际上是同一个变量v**。

```go
func test() {  
    a := []int{1, 2, 3} 
    for _, v := range a {    
        go func() {     
            fmt.Printf("%d ", v)   
        }()  
    }
}
func main() {  
    test()  
    time.Sleep(time.Second)
}
```

正确的写法应该如下：

```go
func test() {  
    a := []int{1, 2, 3} 
    for _, v := range a {    
        go func(i int) {   
            fmt.Printf("%d ", i)  
        }(v)  
    }
}
func main() { 
    test()
    time.Sleep(time.Second)
}
```

## **sync.WaitGroup** 

sync.WaitGroup的功能主要是等待一组gorountine运行结束。它提供了三个方法，分别为Add、Done和Wait。典型的应用场景是一个gorountine初始化sync.WaitGroup，并调用Add设置一个计数器(N)，然后创建若干(N个)goroutine，最后调用Wait等待所有创建的goroutine运行结束；而每个新创建的goroutine在执行结束后调用Done。注意，**一定要在创建需要等待的goroutine之前调用Add设置好计数器**。例如，下面的例子就是错误的用法，

```go
import (  "fmt"  "sync")
func main() {  
    var sg sync.WaitGroup
  	for i := 0; i < 3; i++ {    
        go func() {      
            sg.Add(1)      
            fmt.Println("goroutine running...")      
            sg.Done()    
        }()  
    }
    sg.Wait()
}
```

上面的例子有两种可能的错误结果：

1. 主goroutine在调用Wait时，几个新创建的goroutine都还没来得及调用Add，计数器一直是0，所以Wait并没有阻塞，而是直接返回。
2. 主goroutine在调用Wait时，某个新创建的goroutine刚好正在第一次调用Add，由于并发冲突从而产生panic。

将上面的错误的例子稍微修改一下，正确的写法如下：

```go
func main() {  
    var sg sync.WaitGroup  
    sg.Add(3)
  	for i := 0; i < 3; i++ {    
        go func() {      
            fmt.Println("goroutine running...")      
            sg.Done()    
        }()  
    }
  	sg.Wait()
}
```

另外要注意的是，**当将sync.WaitGroup作为参数传递给一个函数或方法时，一定要传指针，因为sync.WaitGroup是包含sync.noCopy，是禁止拷贝的**。例如，下面就是错误的用法，由于sync.WaitGroup禁止拷贝，所以函数test接收到的sync.WaitGroup副本的计数是0。因为sg.Done()实际上是调用sg.Add(-1)，所以计数值变成负数，从而产生panic。

```go
func test(sg sync.WaitGroup) {  
    fmt.Println("goroutine running...")  
    sg.Done()
}
func main() {  
    var sg sync.WaitGroup  
    sg.Add(3)  
    for i := 0; i < 3; i++ {    
        go test(sg)  
    }  
    sg.Wait()
}
```

正确的写法应该是传指针，如下：

```go
func test(sg *sync.WaitGroup) {  
    fmt.Println("goroutine running...")  
    sg.Done()
}
func main() {  
    var sg sync.WaitGroup 
    sg.Add(3) 
    for i := 0; i < 3; i++ {   
        go test(&sg) 
    }  
    sg.Wait()
}
```

## **sync.Mutex & sync.RWMutex** 

只注意一点，**sync.Mutex和sync.RWMutex作为参数传递时，一定要传指针**。例如，下面的例子用同一个sync.Mutex实例同步两个goroutine的执行，

```go
func test1(m *sync.Mutex) {  
    m.Lock()  
    defer m.Unlock()  
    fmt.Println("test1 begin running...")  
    time.Sleep(time.Second)  
    fmt.Println("test1 exiting...")
}
func test2(m *sync.Mutex) {  
    m.Lock()  
    defer m.Unlock()  
    fmt.Println("test2 begin running...")  
    time.Sleep(time.Second)  
    fmt.Println("test2 exiting...")
}
func main() {  
    var m sync.Mutex  
    go test1(&m)  
    go test2(&m)  
    time.Sleep(4 * time.Second)  
    fmt.Println("main exiting...")
}
```

上面的例子的输出结果为：

```go
test1 begin running...
test1 exiting...
test2 begin running...
test2 exiting...
main exiting...
```

或者：

```go
test2 begin running...
test2 exiting...
test1 begin running...
test1 exiting...
main exiting...
```

## **switch** 

首先，要明确一点，**在golang中不需要在每个case里面加break，因为当某个匹配的case执行完之后，会自动退出switch**。只有明确使用了关键字"fallthrough"才会继续执行一个case中的语句。注意，**只要使用了fallthrough，就会无条件执行下一个case中的语句，哪怕下一个case并不匹配**。例如，下面的例子的输出结果为"3 4"，

```go
func main() {  
    val := 3  
    switch val {  
    case 3:    
        fmt.Println("case 3")    
        fallthrough  
    case 4:    
        fmt.Println("case 4")  
    case 5:   
        fmt.Println("case 5")  
    }
}
```

另外，要注意的是，**fallthrough必须是case代码块的最后一条语句，否则编译会失败**。例如，下面的例子中的用法就是错误的，

```go
switch val {  
case 3:    
    fmt.Print("3 ")    
    fallthrough //invalid    
    fmt.Print("3 ")  
case 4:    
    fmt.Print("4 ")  
}
```

**每一个case后面可能接多个选项**，例如：

```go
switch val {  
case 1，2，3:    
    fmt.Println("1 2 3")  
case 4:    
    fmt.Println("4")  
}
```

前面已经说过了，在switch case中不需要加break。但有些情况下，break还是有用处的。**break的第一个用处就是提前退出case的执行**。例如，下面的例子中，break之后的语句就不会执行。

```go
switch val {  
case 1，2，3:    
    fmt.Println("1 2 3")    
    if val == 3 {        
        break    
    }    
    fmt.Println("1 2")      
case 4:    
    fmt.Println("4")  
}
```

**break另外一个实用的场景就是直接退出外层循环**。例如，下面的例子在val等于3时，直接跳出for循环。

```go
func main() {  
    vals := []int{3, 4}
looplabel:  for _, val := range vals {    
        switch {    
        case val == 3:      
            fmt.Println("3")      
            break looplabel    
        case val == 4:      
            fmt.Println("4")    
        }  
    }
}
```

**switch后面的表达式可以省略，如果省略，那么默认就是一个布尔值true**。例如，下面的例子中，switch后面就没有表达式，默认执行第一个case 表达式为true的分支，最后输出结果为"3"。

```go
func main() {  
    val := 3  
    switch {  
        case val == 3:    
        fmt.Println("3")  
        case val == 4:    
        fmt.Println("4")  
    }
}
```

将上面的例子稍微修改一下（如下），**switch后面虽然有"val:=3;"，但分号后面没有表达式，所以同样默认是一个布尔值true**，所以最后输出结果也是"3"。

```go
func main() {  
    switch val := 3; {  
        case val == 3:    
        fmt.Println("3")  
        case val == 4:    
        fmt.Println("4")  
    }
}
```

将上面的例子再稍微修改一下（如下），**switch后面跟一个布尔值false，那就是执行第一个case表达式为false的分支**，最后输出结果为"4"。

```go
func main() {  
    val := 3  
    switch false {  
    case val == 3:    
        fmt.Println("3")  
    case val == 4:    
        fmt.Println("4") 
    }
}
```

switch还有一种典型的使用场景，那就是type switch。大致格式如下，

```go
switch x.(type) {
    // cases
}
```

一定要注意，**x必须是接口类型，可以是interface{}或者自定义的接口**。例如，下面的例子中，变量x的类型是interface{}，但实际上给它赋了一个整型值，最后输出"int"。

```go
func main() {  
    var val interface{} = 3  
    switch val.(type) {  
        case int:    
        fmt.Println("int")  
        case string:    
        fmt.Println("string")  
    }
}
```

下面的例子中，变量s的静态类型是自定义的接口Interface，而它的动态类型是S2，所以最后输出"S2"。

```go
type Interface interface {  
    Do()
}
type S1 struct{}
func (s S1) Do() {  
    fmt.Println("S1 Do()")
}
type S2 struct{}
func (s S2) Do() {  
    fmt.Println("S2 Do()")
}
func main() {  
    var s Interface = S2{}  
    switch s.(type) {  
        case S1:    
        fmt.Println("case S1")  
        case S2:    
        fmt.Println("case S2")  
    }
}
```

注意，**在type switch中不能使用fallthrough**。

最后，值得一提的是，可以为switch增加一个default分支。注意，**default分支可以出现在switch的任何位置**。

## **select** 

select与switch类似，但记住一点，**select只能用于I/O**。另外，**如果多个case中的I/O都准备就绪，那么会随机选择一个执行**。所以下面例子的输出结果是下面两者之一：

> Received value from ch1: 3 
>
> Received value from ch2: 4

```go
func main() {  
    ch1 := make(chan int, 1)  
    ch2 := make(chan int, 1)  
    ch1 <- 3  
    ch2 <- 4  
    var v int  
    select {  
    case v = <-ch1:    
        fmt.Printf("Received value from ch1: %d\n", v)  
    case v = <-ch2:    
        fmt.Printf("Received value from ch2: %d\n", v)  
    }
}
```

## **struct** 

结构体(struct)是golang中很常用的数据类型。**在给一个struct中的字段单独赋值时，一定不能使用:=，而应该用=**。例如，下面的写法就是错误的，编译通不过，

```go
type student struct {  
    age  int  
    name string
}
func test() (int, error) {  
    return 3, nil
}
func main() {  
    var s student  
    s.age, err := test() //invalid  
    if err != nil {    
        fmt.Println(err)  
    }
}
```

正确的写法应该是：

```go
age, err := test()
s.age = age
```

如果要使其它package中的程序能直接访问struct中的字段，那么字段首字母要大写。例如，下面的代码中，**json库位于包json中，不能直接访问结构体student中的非导出字段**，自然Unmarshal不会成功，最后输出"{0 }"。

```go
import (  
    "encoding/json" 
    "fmt"
)
type student struct {  
    age  int    `json:"age"`  
    name string `json:"name"`
}
func main() {  
    var s student  
    txt := `{    "age": 30,    "name": "ben"  }`
  	json.Unmarshal([]byte(txt), &s)  
    fmt.Println(s)
}
```

所以，正确的做法是将结构体中两个字段的首字母大写，

```go
type student struct {  
    Age  int    `json:"age"`  
    Name string `json:"name"`
}
```

**在一个struct内可以定义内嵌字段，也就是字段只有类型，没有名字；这时在当前struct内就自动拥有了内嵌字段的属性和方法。在当前的struct内可以定义与内嵌字段中同名的属性和方法，从而覆盖内嵌字段中相应属性和方法**。例如，在下面的例子中，结构体student中定义了两个内嵌字段，分别是结构体people和int；同时在student定义了同名的字段"name"以及方法"Eat"，覆盖了people中的对应的属性和方法。程序最后输出为：

> I am people.
>
> Students like icecream.
>
> {30 ben} 82 90 benjamin

```go
type people struct {  
    age  int  
    name string
}
func (p *people) Speak() {  
    fmt.Println("I am people.")
}
func (p people) Eat() {  
    fmt.Println("People like cake.")
}
type student struct {  
    people  
    int  
    score int  
    name  string
}
func (s student) Eat() {  
    fmt.Println("Students like icecream.")
}
func main() {  
    p := people{30, "ben"}  
    s := student{p, 82, 90, "benjamin"}  
    s.Speak()  
    s.people.Eat()  
    fmt.Println(s.people, s.int, s.score, s.name)
}
```

在上面的例子中，因为student的属性"name"和方法"Eat"覆盖了people中的同名属性和方法，所以s.name和s.Eat()都是直接访问/调用student中的属性和方法。如果想访问people中被覆盖的属性和方法，则可以采用下面的形式，

```go
s.people.Eat()
fmt.Println(s.people.name)
```

关于内嵌字段，在golang的标准库里的src/sort/sort.go中，有一个经典的案例。sort.go中定义了一个接口Interface（如下），任何容器只要实现了这个接口，就可以直接调用sort.Sort(h)对容器内的元素进行排序，

```go
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {  
    // Len is the number of elements in the collection.  
    Len() int  
    // Less reports whether the element with  
    // index i should sort before the element with index j.  
    Less(i, j int) bool  
    // Swap swaps the elements with indexes i and j.  
    Swap(i, j int)
}
```

**sort.go中通过对内嵌字段的巧妙运行，很简单就实现了反向排序**。首先，结构体reverse内嵌了上面的Interface；其次，为reverse重新实现了方法Less。

```go
type reverse struct {  
    // This embedded Interface permits Reverse to use the methods of  
    // another Interface implementation.  
    Interface
}
// Less returns the opposite of the embedded implementation's Less method.
func (r reverse) Less(i, j int) bool {  
    return r.Interface.Less(j, i)
}
```

我在自己的**开源项目ahrtr/gocontainer中也利用内嵌字段的实现了同样的功能**，具体参考下面的链接：

```go
https://github.com/ahrtr/gocontainer/blob/master/utils/sort.go
```

最后注意一点，**如果内嵌字段是接口类型，则不能使用指针形式**。例如，下面的写法是错误的，编译通不过，

```go
type reverse struct {  
    *Interface
}
```

## **json.Marshal & json.Unmarshal** 

关于json的使用，已经超出了golang基础知识点的范畴了。这里只记住一点，**下列三种类型不能编码成JSON文本。如果尝试调用json.Marshal处理这三种数据结构，会返回UnsupportedTypeError**。

> channel 
>
> complex 
>
> function

## **接收器(Receiver)** 

在golang中，方法(function)和函数(method)是两个不同的概念；方法是带接收器(receiver)的函数。虽然golang不是面向对象的语言，但这里也体现了面向对象中的**封装**的思想。前一篇文章中讲的，在struct里面内嵌字段的方式则体现了**继承**的思想；golang中的Interface更是**抽象和多态**的体现。

**只有defined type(用关键字type定义的类型)才能用作方法的receiver，并且defined type与method必须定义在同一个包中**。例如，下面的写法就是错误的，编译通不过，因为map不是一个defined type，

```go
func (i map[string]int) DoSomething() {  
    fmt.Println("do somthing")
}
```

正确的写法应该是：

```go
type MyMap map[string]int
func (i MyMap) DoSomething() {  
    fmt.Println("do somthing")
}
```

**receiver既可以是defined type(为了行文简洁，后面用T直接代替)，也可以是指向defined type的指针**。例如，下面的例子中，receiver的类型是结构体S，有两个方法与S绑定，其中Do1()绑定的是S，而Do2()绑定的*S。

```go
type S struct{}
func (s S) Do1() {  
    fmt.Println("S Do1()")
}
func (s *S) Do2() {  
    fmt.Println("S Do1()")
}
```

这里要注意，**既可以使用T类型的变量，也可以使用\*T类型的指针变量调用与T或\*T绑定的任意方法**。例如，下面的例子则是使用S类型的变量调用Do1()和Do2(），

```go
s := S{}  
s.Do1()  
s.Do2()
```

而下面的例子则是使用*S类型的指针变量调用方法，

```go
s := &S{}  
s.Do1()  
s.Do2()
```

实际上，在下面两种情况下，**golang编译器会自动做转换**：

1、方法前面的receiver参数是T，而用*T去调用该方法，golang会自动先将*T转换成T，然后再调用方法；

2、方法前面的receiver参数是*T，而用T去调用该方法，golang会自动先将T转换成&T，然后再调用方法；

**无论是哪种调用组合，只要方法前面的receiver参数是\*T，那么方法内对receiver参数的修改，都会保存下来，调用者会看到这些修改。反之，如果方面前面的receiver参数是T，那么方法只是在receiver的副本上修改**。例如，下面的例子中，只有方法Do2()对receiver参数的修改会被调用者看到。

```go
type S struct {  
    age int
}
func (s S) Do1() {  
    fmt.Println("S Do1()")  
    s.age = 11
}
func (s *S) Do2() {  
    fmt.Println("S Do2()")  
    s.age = 12
}
```

当receiver/method与接口结合起来使用，又稍微有点不同了。例如，下面的例子中定义了一个接口，在接口中定义了两个方法Do1和Do2。结构体S绑定了两个方法，Do1对应的receiver参数是类型S，而Do2对应的receiver参数是类型*S。函数test的参数是Interface类型的变量。

```go
type Interface interface {  
    Do1()  
    Do2()
}
type S struct{}
func (s S) Do1() {  
    fmt.Println("S Do1()")
}
func (s *S) Do2() {  
    fmt.Println("S Do2()")
}
func test(h Interface) {  
    h.Do1()  
    h.Do2()
}
```

这里一定要注意，**调用上面例子中的函数test时，必须传入\*S类型的变量**。例如，下面这种调用方法是错误的，编译通不过，**因为编译器认为S并没有实现接口Interface**，更进一步的原因是S没有实现Interface中的Do2，因为Do2对应的receiver是*S。

```go
  s := S{}  
  test(s)
```

解决办法有两个，第一种解决办法是传*S类型的变量给函数test，如下：

```go
  s := S{}  
  test(&s)
```

另一种解决办法就是将Do2前面的receiver参数改成S，如下：

```go
func (s S) Do2() {  
    fmt.Println("S Do2()")
}
```

以上两种解决方案各有优缺点。**当需要在方法内修改receiver参数时，receiver参数必须用\*T。如果方法内不修改receiver参数，再看receiver包含的数据是否很大，如果很大，receiver参数也尽量采用\*T的形式，否则可以考虑采用T的形式**。

另外，要注意的是，**一个方法与"将receiver参数作为第一个参数的函数"是等价的，它们本质上具有相同的类型，所以可以将方法调用转化为相应的普通函数调用**。例如，下面为结构体S绑定了一个方法Do。

```go
type S struct {  
    name string
}
func (s S) Do(v int) {  
    fmt.Println(s.name, v)
}
```

正常情况下，用类似下面的代码来调用方法Do，

```go
s := S{"ben"}
s.Do(4)
```

或者用指针形式的receiver，也是正确的，

```go
s := &S{"ben"}
s.Do(4)
```

其实还可以将方法调用转换成普通函数的方式，例如，下面的例子将receiver作为第一个参数调用函数f，

```go
s := S{"ben"}
f := S.Do
f(s, 4)
```

或者下面这样也是正确的，

```go
s := S{"ben"}
f := (*S).Do
f(&s, 4)
```

最后捎带提醒一下，**当你为某个defined type实现了方法"String() string"，就一定不要在方法内调用fmt.Sprintf，否则会引起无限递归循环调用**。因为实现了方法"String() string"，就相当于实现了接口Stringer(如下)。那么通过fmt.PrintXXX输出defined type类型的变量时，就会自动调用String()方法，如果String()方法内调用了fmt.Sprintf，那么又会自动递归调用String()方法，从而无限递归循环调用下去，最终会栈溢出。

```go
// Stringer is implemented by any value that has a String method,
// which defines the ``native'' format for that value.
// The String method is used to print values passed as an operand
// to any format that accepts a string or to an unformatted printer
// such as Print.
type Stringer interface {  
    String() string
}
```

## **函数返回值** 

在golang中，**如果一个函数有返回值，可以为每个返回值指定一个变量名，也可以不指定。但是要记住，要么全部都指定，要么都不指定，不允许部分指定**。例如，下面的函数test有两个返回值，而且指定了变量名，分别为a和err，

```go
func test() (a int, err error) {  
    return 4, nil
}
```

上面的例子体现不出指定变量名的用处，稍微修改一下，就明白了。下面的例子与上面的代码作用完全相同，因为函数体内已经为返回变量设置了值，所以return后面不用再列出返回值了。当然，return后面也可以列出返回值，如果列出了，那么最终的返回值就是return后面列出的返回值。

```go
func test() (a int, err error) {  
    a = 4  
    err = nil  
    return
}
```

**带命名的返回值本身并不复杂，但与defer结合起来使用就容易使人迷惑了。记住一个原则，函数执行return的流程是：****先设置返回变量的值，然后再执行defer函数**。例如，下面的例子，当执行到return时，先将5和nil分别设置给返回变量a和err；然后再执行defer函数，执行a++，返回变量a的值变成了6。所以最后返回值是6和nil。

```go
func test() (a int, err error) {  
    defer func() {    
        a++  
    }()
  	return 5, nil
}
```

另外一个容易踩的坑是与变量的作用域相关。例如，下面的例子中，**在for循环内的变量a是一个新定义的临时变量，与返回变量a没有关系**。

```go
func test() (a int, err error) {  
    for i := 0; i < 3; i++ {    
        a := calc()    
        fmt.Println(a)  
    }
  	return
}
```

**当一个返回变量被一个同名的临时变量或常量掩盖时，是不允许在该临时变量的作用域内执行不带返回参数的return**。例如，下面例子中第5行的return不带返回值，这样的写法是错误的，编译通不过；因为返回变量a和err被for循环内的同名临时变量掩盖了。解决办法有两个：1、return后面列出返回值；2、跳出临时变量的作用域之后，再用不带返回值的return。

```go
func test() (a int, err error) {  
    for i := 0; i < 3; i++ {    
        a, err := doSomthing()    
        if err != nil {      
            return    
        }    
        fmt.Println(a, err)  
    }
  	return
}
```

## **包(package)** 

包的主要作用是组织源代码文件。包中有一种特殊的函数init，没有参数，也没有返回值，用来执行初始化的操作。注意，**函数init是在包被import时，自动执行的，不能显示调用**。

同一个包中可以定义多个init函数，**位于同一个源文件中的init函数按照定义的先后顺序执行，不同源文件中的init则没有明确的顺序。但是如果当前包又import了其它包，那么会先执行依赖包中的init函数**。

**如果一个包被import了多次，它的init函数只会执行一次**。

与变量定义之后必须使用一样，import了一个包之后，也必须使用这个包。如果只想利用import一个包产生的副作用(执行init函数)，那么可以采用下划线的形式，例如，下面的例子import了包"time"，但并不打算使用该包，

```go
import (  
    "fmt"  
    _ "time"
)
```

一般情况下，包的名字通常与目录名相同，同一个目录下的所有文件属于同一个包。**但是有一个特例，就是测试源文件允许定义在不同的包，但包名必须是在原包名后加上“_test”**。例如假设有一个包名是"abc"，源文件都在目录"abc"里面；但是测试源文件虽然也在目录"abc"里面，但包名可以是"abc_test"。golang标准库中有很多这样的例子，例如src/runtime/string.go文件位于包runtime中，而src/runtime/string_test.go位于包runtime_test中。**这样带来的一个好处是，在测试程序中只能看到对应的源文件中导出的变量和函数，与实际使用场景完全一样**。

## **new & make** 

new和make都是golang中的预定义标识符，都是golang的内置函数。区别在于new是在运行时为某个类型T的变量分配内存，**返回的是指向新分配内存的指针\*T**。make虽然也用来分配内存，但make只能用来操作slice、map、channel这三种类型，而且**返回值是已经初始化过的值**，而不是指针。

这里附带提一句，**在一个函数中返回一个临时变量的地址之后，只要调用者继续持有该指针，不用担心该变量的空间会被回收**。例如，下面的例子中，函数test返回一个结构体临时变量的指针，只要调用者继续持有该指针，这个结构体的空间就不会被回收，

```go
type S struct {  
    age  int  
    name string
}
func test() *S {  
    return &S{    
        age:  30,    
        name: "ben",  
    }
}
```

## **unsafe.Pointer** 

golang中，不同类型的指针之间不能进行类型转换。例如，下面的写法是错误的，

```go
fv := 3.0
var v1 *int64 = (*int64)(&fv)
```

但是借助于unsafe.Pointer却可以做到不同类型的指针相互转换。但是要注意，**使用包unsafe的应用程序可能是不可移植的，而且不受golang 1.x兼容性保证**。所以建议不要轻易使用。

将上面错误的例子稍微修改一下，借助于unsafe.Pointer就可以将*float64转换成*int64，

```go
fv := 3.0
var v1 *int64 = (*int64)(unsafe.Pointer(&fv))
```

**包unsafe的另一个典型的应用场景是修改struct的字段**。例如，下面的例子，通过结合使用unsafe.Pointer、unsafe.Offsetof以及uintptr，完成了对结构体成员的修改，

```go
type S struct {  
    age  int  
    name string
}
func main() {  
    s := S{20, "ben"}
  	ps := unsafe.Pointer(&s)
  	pAge := (*int)(ps)  
    *pAge = 30
  	pName := (*string)(unsafe.Pointer(uintptr(ps) + unsafe.Offsetof(s.name)))  
    *pName = "alice"
  	fmt.Println(s)
}
```

## **sort & heap** 

golang标准库中的两个包sort和heap提供了对排序和堆的支持。sort中定义了下面的接口，**任何容器只要实现了这个接口，就可以直接调用sort.Sort(h)进行排序，或者调用sort.Sort(sort.Reverse(h))进行反向排序。**

```go
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {  
    // Len is the number of elements in the collection.  
    Len() int  
    // Less reports whether the element with  
    // index i should sort before the element with index j.  
    Less(i, j int) bool  
    // Swap swaps the elements with indexes i and j.  
    Swap(i, j int)
}
```

**sort为slice提供了一个更便捷的排序函数sort.Slice。应用层只需要提供Less函数的实现**。例如，下面的例子就是一个使用sort.Slice的例子，最后输出为：

> [1 2 3 4]

```go
func main() {  
    s := []int{1, 4, 2, 3}  
    sort.Slice(s, func(i, j int) bool {    
        return s[i] < s[j]  
    })  
    fmt.Println(s)
}
```

包heap中定义了下面的接口，

```go
type Interface interface {  
    sort.Interface  
    Push(x interface{}) // add x as element Len()  
    Pop() interface{}   // remove and return element Len() - 1.
}
```

**任何容器只要实现了上面的接口，便可直接调用包heap中的下列函数**，

```go
func Init(h Interface) 
func Push(h Interface, x interface{})
func Pop(h Interface) interface{}
func Remove(h Interface, i int) interface{} 
func Fix(h Interface, i int)
```

下面是一个完整的使用heap的例子，最后输出结果为：

> 3 4 5 6 7 9

```go
type S struct {  
    items []interface{}
}
func (s *S) Len() int {  
    return len(s.items)
}
func (s *S) Less(i, j int) bool {  
    return s.items[i].(int) < s.items[j].(int)
}
func (s *S) Swap(i, j int) {  
    s.items[i], s.items[j] = s.items[j], s.items[i]
}
func (s *S) Push(v interface{}) {  
    s.items = append(s.items, v)
}
func (s *S) Pop() interface{} {  
    n := s.Len()  
    ret := s.items[n-1]  
    s.items = s.items[:(n - 1)]  
    return ret
}
func main() {  
    s := S{    
        items: []interface{}{4, 9, 3, 5, 7},  
    }  
    heap.Init(&s)  
    heap.Push(&s, 6)  
    for s.Len() > 0 {    
        fmt.Printf("%d ", heap.Pop(&s))  
    }
}
```

上面的例子不算太复杂，就不做过多的解释了。**我在自己的开源项目gocontainer中对golang标准库中的heap的实现做了一些改动，里面有更详细的关于heap的介绍。**具体请参考：

```go
https://github.com/ahrtr/gocontainer/blob/master/utils/heap.go
```

不管是sort还是heap，golang标准库只是提供了标准的接口以及函数，供应用层使用。**应用层需要实现标准库定义的接口，并且需要自己管理数据结构，但是可以利用标准库提供的函数来操作这些数据结构**。

