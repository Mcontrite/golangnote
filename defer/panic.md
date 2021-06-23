## Go 错误处理与异常

原创 [darlingtangli](https://studygolang.com/user/darlingtangli) 2018-11-05 19:01:31  [原文地址](http://litang.me/post/Golang-error-and-panic/)

+++

Golang 中的错误处理的哲学和 C 语言一样，函数通过返回错误类型(error)或者 bool 类型(不需要区分多种错误状态时)表明函数的执行结果，调用检查返回的错误类型值是否是 nil 来判断调用结果。

## error

Golang 中内置的错误类型 error 是一个接口类型，自定义的错误类型也必须实现为 error 接口，这样就可以通过调用 Error() 获取到具体的错误信息而不用关心错误的具体类型。

标准库的 `fmt.Errorf` 和 `errors.New` 可以方便的创建 error 类型的变量。

```go
type error interface {
    Error() string
}
```

C 语言限制函数只能有一个返回值，函数的执行结果和执行成功时需要返回的信息都放到这个返回值里，具体的错误信息需要调用额外的接口获取。比如 C 标准库函数读取文件的函数 read 返回 -1 时表示读取错误，具体错误信息需要通过 errno 获取，返回 0 时，表示 EOF ，返回正整数时，表示成功读取到的字节数。

Golang 的多返回值语法糖避免了这种方式带来的不便，错误值一般作为返回值列表的最后一个，其他返回值是成功执行时需要返回的信息。

## error 处理

为了避免错误处理时过深的代码缩进：

```go
if err != nil {
    // error handling
} else {
    // normal code
}
```

推荐在发生错误时立即返回：

```go
if err != nil {
    // error handling
    return // or continue, etc.
}
// normal code
```

虽然这种错误处理方式代码写起来会有一些冗长，但 Golang 风格的代码应该尽量使用这种方式。

## 预定义全局错误值

下面的示例代码中，当需要返回错误时，每次都调用 `errors.New()` 返回一个 error 对象：

```go
func doStuff() error {
    if someCondition {
        return errors.New("no space left on the device")
    } else {
        return errors.New("permission denied")
    }
}
```

当需要对特定错误进行处理时，这种方法不方便对错误值进行等值判断。只能用 `Error()` 取出字符串比较，这样即不优雅也容易出现拼写错误。最佳的做法是预定义全局的错误值：

```go
var ErrNoSpaceLeft = errors.New("no space left on the device")
var ErrPermissionDenied = errors.New("permission denied")

func doStuff() error {
    if someCondition {
        return ErrNoSpaceLeft
    } else {
        return ErrPermissionDenied 
    }
}
```

这样错误值判断就方便多了：

```go
if err == ErrNoSpaceLeft {
    // handle this particular error
}
```

标准库中也预定义了一些错误值 ，最常用的就是 `io.EOF` 。

## 自定义错误类型

HTTP 表示客户端的错误状态码有几十个。如果为每种状态码都预定义相应的错误值，代码会变得很繁琐：

```go
var ErrBadRequest = errors.New("status code 400: bad request")
var ErrUnauthorized = errors.New("status code 401: unauthorized")
// ...
```

最佳的最法是自定义一种错误类型，并且实现 `Error()` 方法(满足 error 接口定义)：

```go
type HTTPError struct {
    Code        int
    Description string
}

func (h *HTTPError) Error() string {
    return fmt.Sprintf("status code %d: %s", h.Code, h.Description)
}
```

这种方式下进行等值判断时需要转成具体的自定义类型然后取出 Code 字段判断：

```go
func request() error {
    return &HTTPError{404, "not found"}
}

func main() {
    err := request()
    if err != nil {
        // an error occured
        if err.(*HTTPError).Code == 404 {
            // handle a "not found" error
        } else {
            // handle a different error
        }
    }
}
```

自定义错误类型可以提供更多特定的错误信息，标准库中也有一个这样的例子 `os.PathError`。

## Panic 

Golang 新手很容易把 panic 当成其他语言的异常(exception)导致 panic 的滥用。为什么 Golang 没有提供类似其他语言的异常机制的原因：主要是 try / catch 的方式处理错误过于复杂。使用 panic / recover 的机制也可以达到类似 try / catch 的效果，但是要特别谨慎的使用。

panic / recover 和 try / catch 机制最大的不同在于控制流程上的区别。

1. try / catch 机制控制流作用在 try 代码块内，代码块执行到异常抛出点(throw)时，控制流跳出 try 代码块，转到对应的 catch 代码块，然后继续往下执行。
2. panic / recover 机制控制流则作用在整个 goroutine 的调用栈。当 goroutine 执行到 panic 时，控制流开始在当前 goroutine 的调用栈内向上回溯(unwind)并执行每个函数的 defer 。如果 defer 中遇到 recover 则回溯停止，如果执行到 goroutine 最顶层的 defer 还没有 recover ，运行时就输出调用栈信息然后退出。

所以如果要使用 recover 避免 panic 导致进程挂掉，recover 必须要放到 defer 里。recover 调用的位置如果不对是无法将 panic 恢复的，为了避免过于复杂的代码，最好不要使用嵌套的 defer ，并且 recover 应该直接放到 defer 函数里直接调用。

## Panic 使用场景

### 发生严重错误必须让进程退出

这里“严重”判断的标准是错误无法恢复，导致程序无法执行或者继续执行会发生不可预期的行为。有点类似如 C 语言的 assert 机制。

标准库中有些函数也是使用这种机制，比如 `regexp.MustCompile`。当传入的参数不是一个合法的正则表达式时，继续执行已经没有任何意义。

另外一些场景，比如程序启动时依赖的数据库不存在或者依赖的配置不可读取，这个时候如果继续执行可能会导致发生不可预期的行为，这个时候使用 panic 让进程直接退出将问题暴露反而是更可取的做法。

非“严重”的错误比如客户端不合法的请求参数应该返回 error 然后让调用者去处理而不是使用 panic 让进程退出。

### 快速退出错误处理

也就是上文中提到的模拟 try / catch 的行为。

大多数情况下错误处理应该使用 error 机制，但有时候函数调用栈很深，逐层返回错误可能需要写很多冗余代码，这个时候可以使用 panic 让程序的控制流直接跳到顶层的 recover 处来处理错误。这种场景要特别注意在包内的 panic 必须在包内就要 recover 。让 panic 跨包传递可能会导致更复杂的问题，所以包的导出函数不应该产生 panic (上述 1 中的场景除外)。

## 总结

本文简要介绍了 Golang 的错误处理与异常机制。error 是 Golang 中错误处理的主要方式，Golang 程序应该尽量使用 error 来进行错误处理。

1. 当需要对 error 进行等值判断以针对特定的错误类型进行处理时，预定义全局的错误值是比较优雅的方式。
2. 当需要提供额外的错误信息时，可以自定义错误类型，但至少要实现 `Error()` 方法以满足 error 的定义。
3. Golang 虽然也有类似其他语言的异常的 panic ，但是使用场景很有限，如非必要不要使用。