## ﻿golang channel 源码解析

channel一个类型管道，通过它可以在goroutine之间发送和接收消息。它是Golang在语言层面提供的goroutine间的通信方式。

众所周知，Go依赖于称为CSP（Communicating Sequential Processes）的并发模型，通过Channel实现这种同步模式。Go并发的核心哲学是不要通过共享内存进行通信; 相反，通过沟通分享记忆。

Go中那句经典的话：`Do not communicate by sharing memory; instead, share memory by communicating.`的具体实现就是利用channel把数据从一端copy到了另一端。

## hchan

```go
type hchan struct {
    qcount   uint           // chan 里元素的数量
    dataqsiz uint           // chan 底层循环数组的长度
    buf      unsafe.Pointer // 指向底层循环数组的指针, 只针对缓存型 chan. 内存肯定连续
    elemsize uint16 // chan 中元素大小
    closed   uint32 // chan 是否被关闭的标志
    elemtype *_type // chan 在元素类型 (element type)
    sendx    uint   // 已发送元素在循环数组中的索引 (send index)
    recvx    uint   // 已接收元素在循环数组中的索引 (receive index)

    // 注意: recvq, sendq 都表示被阻塞的 goroutine, 这些 goroutine 由于尝试读取 channel
    // 或者向channel 发送数据而被阻塞 
    recvq    waitq  // 等待接收的 goroutine 队列 (list of recv waiters)
    sendq    waitq  // 等待发送的 goroutine 队列 (list of send waiters)

    // lock保护了hchan中的所有字段, 以及被此 channel blocked 的 sudog 中的几个字段.
    // 不要在持有此锁的同时更改另一个G的状态(特别是, 处于ready状态的G), 因为这可能会因堆栈收缩
    // 而导致死锁.
    lock mutex
}
```

- **qcount** uint // 当前队列中剩余元素个数
- **dataqsiz** uint // 环形队列长度，即buffer缓冲区的大小，也就是`make(chan T, N)`中的N
- **buf** unsafe.Pointer // 环形队列指针，是带缓冲的channel(buffered channel)中用来保存数据的循环队列
- **elemsize** uint16 // channel中单个元素的大小
- **closed** uint32 // 表示channel是否关闭。0->打开，1->关闭
- **elemtype** *_type // 元素类型，用于数据传递过程中的赋值；
- **sendx** uint和**recvx** uint表示循环队列接受和发送数据的下标，是环形缓冲区的状态字段，它指示缓冲区的当前索引 - 支持数组，它可以从中发送数据和接收数据。
- **recvq** waitq // 等待读消息的goroutine队列，保存读取数据而阻塞的`Goroutine`
- **sendq** waitq // 等待写消息的goroutine队列，保存写入数据而阻塞的`Goroutine`
- **lock** mutex // 互斥锁，为每个读写操作锁定通道，因为发送和接收必须是互斥操作。

`recvx`和`sendx`是根据循环链表`buf`的变动而改变的。 至于为什么channel会使用循环链表作为缓存结构，我个人认为是在缓存列表在动态的`send`和`recv`过程中，定位当前`send`或者`recvx`的位置、选择`send`的和`recvx`的位置比较方便吧，只要顺着链表顺序一直旋转操作就好。缓存中按链表顺序存放，取数据的时候按链表顺序读取，符合FIFO的原则。

channel 的整体结构图：

![img](https://i6448038.github.io/img/channel/hchan.png)

## waitq 

是 `sudog` 的一个双向链表, 而 `sudog` 实际上是对 goroutine 的一个封装.

```go
type waitq struct {
    first *sudog
    last  *sudog
}
```

## sudog

sudog 表示 goroutine

```go
type sudog struct {
    // The following fields are protected by the hchan.lock of the
    // channel this sudog is blocking on. shrinkstack depends on
    // this for sudogs involved in channel ops.

    g *g

    // isSelect indicates g is participating in a select, so
    // g.selectDone must be CAS'd to win the wake-up race.
    isSelect bool
    next     *sudog
    prev     *sudog
    elem     unsafe.Pointer // data element (may point to stack)

    // The following fields are never accessed concurrently.
    // For channels, waitlink is only accessed by g.
    // For semaphores, all fields (including the ones above)
    // are only accessed when holding a semaRoot lock.

    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel
}
```

## 常量

```go
const (
    maxAlign  = 8
    // 在 64 位系统上 hchan 大小为 96, hchanSize=96
    // 在 32 位系统上 hchan 大小为 48, hchanSize=48
    hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
)
```

## 相关函数

```go
// 获取buffer上第i个slot的位置. 建立在内存连续的基础上
func chanbuf(c *hchan, i uint) unsafe.Pointer {
    return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}

func (c *hchan) raceaddr() unsafe.Pointer {
    // 将 cahn 上的读取和写入操作视为在此地址发生.
    // 避免使用qcount或dataqsiz的地址. 
    // 因为内置的len()和cap()会读取这些地址, 并且我们不希望它们与close()之类的操作竞争.
    return unsafe.Pointer(&c.buf)
}
```

## 创建

创建channel实际上就是在内存的堆中实例化了一个`hchan`的结构体，并返回一个ch指针，我们使用过程中channel在函数之间的传递都是用的这个指针，这就是为什么函数传递中无需使用channel的指针，而直接用channel就行了，因为channel本身就是一个指针。

一般而言, 使用 `make` 创建一个能收能发的通道:

```go
// 无缓冲通道
ch1 := make(chan int)

// 有缓冲通道
ch2 := make(chan int, 10)
```

通过汇编分析, 最终创建 chan 的函数是 `makechan`

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // 编译器的检查, 元素的大小, 以及基本的校验
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }

    // 计算 elem.size 与 size 的乘积是否溢出
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
        panic(plainError("makechan: size out of range"))
    }

    // 当存储在buf中的元素不包含指针时, hchan将不包含GC感兴趣的指针. buf指向相同的分配, elemtype是可持久化的.
    // sudog 从它们自己的线程中引用, 因此无法将其收集.
    var c *hchan
    switch {
    case mem == 0:
        // elem.size*size 结果为0, 要么是 elem.size 为0, 要么是创建的size为0
        // 此时只分配 hchan 大小的内存.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = c.raceaddr()
    case elem.ptrdata == 0:
        // elem 当中不包含指针. hchan和buf的地址是连续的. 此时分配 hchanSize + mem
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize) // buf 的地址的在 hchan 之后
    default:
        // elem 包含指针, 两次内存操作. hchan 和 buf的地址是不连续的
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
    }

    return c
}
```

例如, 当我们创建一个 size 为 6, 元素类型为 int 的 chan, 其内存结构如下:

![img](https://pic3.zhimg.com/80/v2-2c919e805310cca25cedb631922dbbde_720w.jpg)

## 发送

```cpp
// 如果 block 不为 true, 则该 protocol 不会休眠
// 当关闭与休眠有关的chan时, 可以使用 g.param == nil 唤醒休眠. 
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // channnl 为nil时, 当进行非阻塞状况下(即 block 为 false), 立即返回, 否则永远休眠当前的 goroutine 到永远.
    if c == nil {
        if !block {
            return false
        }
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // 开启了 -race 调试 
    if raceenabled {
        racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
    }

    // 快速判断: 在非阻塞模式下, 快速检测到失败, 不用获取锁, 快速返回(false)
    // 
    // 如果 chan 未关闭且 chan 没有多余的缓冲空间. 这可能是:
    // 1. chan 是非缓冲型的, 且等待接收队列里没有 goroutine
    // 2. chan 是缓冲型的, 但缓冲队列已经装满了元素
    // 
    // c.closed == 0, chan 未关闭
    // c.dataqsiz == 0 && c.recvq.first == nil, 非缓冲型 chan, 且 "接收队列(recvq)" 当中没有元素
    // c.dataqsiz > 0 && c.qcount == c.dataqsiz, 缓存型 chan, 且 "缓冲队列(buf)" 已满
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks() // 记录当前的 cpu 的 ticket
    }

    // 加锁状况下操作.
    lock(&c.lock)

    // 当前的 chan 已经关闭, 那当然就无法发送了!!
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // 从接收队列当中寻找一个接收者, 直接将当前的内容发送给这个接收者, 绕过缓存区
    if sg := c.recvq.dequeue(); sg != nil {
        // 找到了等待的接收者. 可以绕过通道缓冲区(如果有)将要发送的值直接发送到接收者.
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // 对于缓冲型 chan, 还存在缓冲空间
    if c.qcount < c.dataqsiz {
        // 存在可用的缓冲区, 将发送元素入队列
        qp := chanbuf(c, c.sendx)

        // 开启 -race
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }

        // 数据移动
        typedmemmove(c.elemtype, qp, ep)

        // 修正 sendx 和 qcount 变量
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    // 当前没有合适的缓冲区来存储发送的元素. 这个时候需要根据 "是否是阻塞发送" 来做出抉择
    // 如果是 "阻塞发送", 那么就需要休眠当前的 goroutine
    // 如果是 "非阻塞发送", 那么就快速失败返回(false) 
    if !block {
        unlock(&c.lock)
        return false
    }

    // 阻塞当前的 goroutine 
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }

    // 在"分配elem"和"将mysg入队到gp.waitcopy可以找到它的地方" 之间没有堆栈拆分.
    mysg.elem = ep // 
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg) // 将当前的 sudog 入队列 sendq

    // 带 lock 休眠的休眠操作, 意味着在 goparkunlock 需要解锁 lock
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)

    // 确保 "发送的值" 保持活跃状态, 直到接收者将其复制出来. 也就是延长发送的值的生存周期, 防止提前被 GC 回收
    // sudog具有指向堆栈对象的指针, 但是sudog不被视为stack跟踪器的root.
    KeepAlive(ep)

    // 唤醒后的一些检查和清理操作
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil

    // 注意: 这里的 gp.param 的值是 mysg
    if gp.param == nil {
        // 唤醒的以后, 发现 chan 已经关闭了, 那么发送肯定会存在问题
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
// send 是将发送方发送的ep值复制到接收方sg中(跳过缓冲区). 然后将接收方sg唤醒, 继续执行后续的操作.
//
// chan c 必须为空且已锁定. 使用unlockf()在发送后解锁c.
// sg 必须已经从 recvq 中出队.
// ep 必须为非nil, 并且指向堆或调用者的堆栈.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 开启了 -race 追踪 
    if raceenabled {
        if c.dataqsiz == 0 {
            racesync(c, sg)
        } else {
            qp := chanbuf(c, c.recvx)
            raceacquire(qp)
            racerelease(qp)
            raceacquireg(sg.g, qp)
            racereleaseg(sg.g, qp)
            c.recvx++
            if c.recvx == c.dataqsiz {
                c.recvx = 0
            }
            c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
        }
    }

    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep) // 直接发送
        sg.elem = nil
    }

    gp := sg.g
    unlockf()

    // 唤醒正在休眠的接收者
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}
```

直接发送

```go
// 仅在 "一个正在运行的goroutine写入到另一个正在运行的goroutine的stack" 的操作是在 "无缓冲" 或 "空缓冲" chan 上
// 发送和接收.
// 
// GC假定 stack 写入仅在goroutine运行时发生, 并且仅由该goroutine完成. 
// 使用写屏障足以弥补违反该假设的缺点, 但是写屏障必须起作用.
// typedmemmove 将调用 bulkBarrierPreWrite, 但是目标字节不在 heap 中, 因此这无济于事. 
// 因此这里安排调用 memmove 和 typeBitsBulkBarrier.
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    // src在我们的stack上, dst是另一个stack上的插槽.

    // 一旦我们从sg中读出了sg.elem, 如果目标 stack 被复制(缩小), 它将不再被更新.
    // 因此, 请确保在 read 和 use 之间没有任何抢占点.
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)

    // 不需要go写屏障检查, 因为dst始终是Go内存.
    memmove(dst, src, t.size)
}
```

> **typedmemmove()** VS **memmove()**
>
> typedmemmove() 是将类型 _type 的对象从 src 拷贝到 dst, src和dst都是unsafe.Pointer, 这当中涉及到了内存操作. 由于GC的存在, 在拷贝前, 如果 _type 包含指针, 需要开启写屏障 bulkBarrierPreWrite. 然后调用 memmove() 进行内存 拷贝的操作. 
>
> memmove() 直接是进行内存移动, 使用的是汇编实现的, 代码位置在 `src/runtime/memmove_amd64.s`. 当然了, 这个操作 本身就是单纯的内存操作, 不涉及其他任何内容.

------

> **bulkBarrierPreWrite(dst, src, size uintptr)**
>
> **bulkBarrierPreWrite** 使用**[dst, dst + size)** 的 pointer/scalar 信息对内存范围**[src, src + size)** 中的每个指针 slot 执行写屏障. 这会在执行 memmove 之前执行必要的写屏障.
> src, dst和size必须是 pointer-aligned. 范围 **[dst，dst + size)** 必须位于单个对象内. 它不执行实际的写操作.
>
> 作为一种特殊情况, src == 0 表示正在将其用于memclr. bulkBarrierPreWrite将为每个写障碍的src传递0.
>
> 调用方应在调用memmove(dst,src,size) 之前立即调用 bulkBarrierPreWrite. 此函数标记为 nosplit, 以避免被抢占; GC 不得停止memmove和执行屏障之间的goroutine. 调用者还负责go指针检查, 这可能是将Go指针写入非Go内存中.
>
> 对于分配的内存中完全不包含指针的情况(即src和dst不包含指针), 不维护 pointer bitmap; 通常, 通过检查 typ.ptrdata/ bulkBarrierPreWrite 的任何调用者都必须首先确保基础分配包含指针.

> **typeBitsBulkBarrier(typ \*_type, dst, src, size uintptr)**
>
> **typeBitsBulkBarrier** 对将由 memmove 使用 bitmap 类型定位这些指针slot的每个指针从**[src, src+size)** 复制到 **[dst, dst+size)** 的每个指针执行写屏障.
>
> 类型typ必须与[src, src+size) 和 [dst, dst + size) 完全对应. dst, src和size必须是 pointer-aligned. 类型typ必须具有plain bitmap, 而不是GC程序. 此功能的唯一用途是在 chan 发送中, 并且64kB chan元素大小限制为我们解决了这一问题.
>
> 不能被抢占, 因为它通常在 memmove 之前运行, 并且GC必须将其视为原子动作.

发送数据时先判断channel类型，如果有缓冲区，判断channel是否还有空间，然后从等待channel中获取等待channel中的接受者，如果取到接收者，则将对象直接传递给接受者，然后将接受者所在的go放入P所在的可运行G队列,发送过程完成，如果未取到接收者，则将发送者enqueue到发送channel，发送者进入阻塞状态，有缓冲的channel需要先判断channel缓冲是否还有空间，如果缓冲空间已满，则将发送者enqueue到发送channel，发送者进入阻塞状态如果缓冲空间未满，则将元素copy到缓冲中，这时发送者就不会进入阻塞状态，最后尝试唤醒等待队列中的一个接受者。

1. `lock`锁住整个`channel`结构体
2. 确定写操作。尝试从`recvq`中取出一个正在等待的goroutine，然后直接将数据写在里面。
3. 如果`recvq`为空，确定`buffer`是否有空间。如果有，从当前goroutine复制数据到缓冲区。
4. 如果缓冲区没有空间，将数据保存在当前执行的goroutine，然后将这个goroutine放在`sendq`队列中，然后这个goroutine 的执行暂停。缓冲区不足的channel，或者没有缓存的channel，数据会保存在`sudog`结构体的`elem`
5. 写入完成释放锁。

注意几个属性buf、sendx、lock的变化，流程图：

![img](https://segmentfault.com/img/remote/1460000019204884)

## 接收

接收操作有两种, 一种带 "ok", 反应 chan 是否关闭; 一种不带 "ok", 这种写法, 当收到相应类型为零值时无法知道是真实发送者发送的值, 还是 chan 被关闭后, 返回接收者的默认类型的零值. 两种写法, 都有各种使用的场景.

编译器对于上述两种写法的处理最终会转换成源码里的两个函数:

```go
// 不带"ok"类型, 例如 val := <-ch
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

// 带"ok"类型, 例如 val, ok := <-ch
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}
```

最终的处理函数都是 `chanrecv`.

```go
// chanrecv在chan c上接收并将接收到的数据写入 ep.
// ep可能为nil, 在这种情况下, 接收到的数据将被忽略.
// 如果 block == false 并且 没有可用元素, 则返回(false,false).
// 否则, 如果 c 关闭, 则*ep为零并返回(true, false).
// 否则, 用一个元素填充 *ep并返回(true, true)
// ep 必须是非nil且指向heap或调用者的stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: 不需要检查ep, 因为它始终在stack中, 或者是反射所分配的新内存.
    // 当前的 chan 为nil, 在非阻塞的状况下, 立即返回 (false,false), 否则阻塞当前的 goroutine 到永远
    if c == nil {
        if !block {
            return
        }
        // 当前的 goroutine 永远休眠下去
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    // 快速判断: 在非阻塞模式下, 快速检测到失败, 不用获取锁, 快速返回(false, false)
    //
    // 当我们观察到 chan 没准备好接收:
    // 1. 非缓冲型, 等待发送队列 sendq 里没有 goroutine 在等待
    // 2. 缓冲型, 在缓冲队列 buf 当中没有元素
    // 
    // 之后, 又观察到 closed == 0, 即 chan 关闭了.
    // 由于 chan 不可能被重复打开, 所以前一个观察的时候 chan 也是未关闭的,  因此在这种状况下直接接收失败, 
    // 返回 (fasle, false)
    //
    // 这里操作顺序在这里很重要: 在进行近距离追踪时, 反转操作可能导致错误的行为.
    //
    // c.dataqsiz == 0 && c.sendq.first == nil, 非缓冲型chan, 当前的 "发送队列(sendq)" 为空
    // c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0 缓冲型chan, 当前的 "缓冲队列(buf)" 为空
    // atomic.Load(&c.closed) == 0, chan 已关闭
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        return
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 加锁
    lock(&c.lock)

    // 当前 chan 关闭的状况下, 并且"缓存队列(buf)"里面没有元素. 
    // 也就是说, 即使是关闭状态, 对于缓冲型的 chan, "缓冲队列(buf)" 里面的元素依旧可以被接收到(qcount>0).
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep) // 根据类型清理相应的内存
        }
        return true, false
    }

    // 阻塞的 "发送队列(sendq)" 当中存在 sender. 则说明当前的缓冲队列已满.
    // 可能的状况:
    // 1. 非缓冲型的 chan
    // 2. 缓冲型的 chan, 但 buf 已经满了
    // 针对1, 直接内存拷贝
    // 针对2, 接收到 buf 的头元素(recvx), 并将发送者的元素放到 buf 的尾部(sendx) (这里sendx和recvx其实值是
    // 一样的), 然后唤醒休眠的发送者.
    if sg := c.sendq.dequeue(); sg != nil {
        // 详细参考 recv 当中的解释
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // 缓冲型, 当前的缓存队列存在元素
    if c.qcount > 0 {
        qp := chanbuf(c, c.recvx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }

        //  将 qp 的数据拷贝到 ep
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp) // 清除 qp 内容

        // 修正 recvx 和 qcount 的值
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    // 当前缓队列没有任何元素, 隐含着 "发送队列(sendq)为空".
    // 非阻塞状况下(block为false), 快速返回 (false, false)
    // 阻塞状况下(block为true), 休眠当前的 goroutine
    if !block {
        unlock(&c.lock)
        return false, false
    }

    // 休眠当前的 goroutine
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }

    mysg.elem = ep // 保存接收数据的地址
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 将当前的 sudog 入接收队列 recvq

    // 持有锁的状况下休眠
    goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

    // 休眠被唤醒的后需要的验证和清理工作
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil // gp.param 的值应该是 mysg. 只有在关闭的时候这个值才被设为 nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
// recv 在 full chan(缓冲通道已满或者非缓冲通道) c上处理接收操作.
// 包含2个部分:
// 1) 由发送方 sg 发送的值被放入通道, 并且唤醒发送方以继续执行其后续的逻辑.
// 2) 接收方接收到的值(当前G)被写入 ep.
// 对于同步通道, 两个值相同.
// 对于异步通道, 接收方从通道缓冲区获取其数据, 而发送方的数据放入通道缓冲区.
//
// chan c 必须已满并且已锁定. recv用 unlockf 函数解锁c.
// sg 必须已经从 sendq 中出队列.
// ep 必须为非nil, 并且指向 heap 或调用者的stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
        // 非缓冲型通道, 或者同步通道.
        // 设置 -race
        if raceenabled {
            racesync(c, sg)
        }
        if ep != nil {
            recvDirect(c.elemtype, sg, ep) // 直接从 sender 复制数据.
        }
    } else {
        // 缓冲型通道, 或者异步通道
        // 
        // 先获取 recvx 对应的 slot, 
        // 先 slot 当中读取数据到 ep 当中, 然后将 sender 发送的数据再插入到 slot 当中.
        // 由于缓冲队列是满的, 所以经过上述的操作之后, 缓冲队列依旧是满的, 但是 recvx 需要向前移动一位(保证chan的元素
        // 都是先进先出的), 而且 sendx 也是需要向前移动一位. 但是 sendx 和 recvx 的值相等.

        // 获取 slot 的位置
        qp := chanbuf(c, c.recvx)

        // copy data from "buf" to "receiver"
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        // copy data from "sender" to "buf"
        typedmemmove(c.elemtype, qp, sg.elem) 

        // 更新 recvx 和 sendx
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }

    // sg 清理工作
    sg.elem = nil
    gp := sg.g
    unlockf()

    // 唤醒对应的发送者
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1)
}
// 原理和 sendDirect 类似
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
    src := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
    memmove(dst, src, t.size)
}
```

`recvq` 的数据结构如下:

![img](https://segmentfault.com/img/remote/1460000019204888)

从整体上来看一下此时 chan 的状态:

![img](https://pic3.zhimg.com/80/v2-6083606a9ec361586e27ffe72e0028f6_720w.jpg)

> G1 和 G2 被挂起了, 状态是 `WAITING`.

接收channel与发送类似首先也是判断channel的类型，然后如果是有缓冲的channel就判断缓冲中是否有元素，接着从channel中获取接受者，如果取到，则直接从接收者获取元素，并唤醒发送者，本次接收过程完成，如果没有取到接收者，阻塞当前的goroutine并等待发送者唤醒，如果是拥有缓冲的channel需要先判断缓冲中是否有元素，缓冲为空时，阻塞当前goroutine并等待发送者唤醒，缓冲如果不为空，则取出缓冲中的第一个元素，然后尝试唤醒channel中的一个发送者

1. 加锁；
2. channel关闭&数据为空，返回零值；
3. 如果有挂在sendq的发送者，从环形缓存拿到第一个数据，然后帮发送者将数据写入环形缓存的末尾；和发送时绕过缓存不同，保证消息FIFO，避免缓存的数据被饿死；
4. 从环形缓存中接收数据；
5. 数据为空，挂在recvq上；被唤醒，收尾工作；

1、先获取channel全局锁

2、尝试sendq从等待队列中获取等待的goroutine，

3、 如有等待的goroutine，没有缓冲区，取出goroutine并读取数据，然后唤醒这个goroutine，结束读取释放锁。

4、如有等待的goroutine，且有缓冲区（此时缓冲区已满），从缓冲区队首取出数据，再从sendq取出一个goroutine，将goroutine中的数据存入buf队尾，结束读取释放锁。

5、如没有等待的goroutine，且缓冲区有数据，直接读取缓冲区数据，结束读取释放锁。

6、如没有等待的goroutine，且没有缓冲区或缓冲区为空，将当前的goroutine加入**recvq**排队，进入睡眠，等待被写goroutine唤醒。结束读取释放锁。

流程图：

![img](https://segmentfault.com/img/remote/1460000019204887)

## 关闭

```go
func closechan(c *hchan) {
    // 当前的 chan 为 nil. 关闭自然会报错
    if c == nil {
        panic(plainError("close of nil channel"))
    }
    lock(&c.lock)
    // 重复关闭一个 chan
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }
    // 设置了 -race 追踪
    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
        racerelease(c.raceaddr())
    }
    // 关闭
    c.closed = 1
    var glist gList // 将所有处于阻塞状态的 g 串在一起. 链接点是 g.schedlink
    // 释放所有的 recvq
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem) // 清除 sg 当中的元素值
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil // 关闭的标志之一
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp) // 将 goroutine 存入 glist
    }
    // 释放所有的 sendq (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp) // 将 goroutine 存入 glist
    }
    unlock(&c.lock)
    // 唤醒 glist 当中所有的 gp. 这样做的好处是减少锁定的时间
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```

## Select原理

特点

1. 可以在channel上进行非阻塞的收发操作；
2. 遇到多个channel同时响应时，随机选择case执行，避免饥饿；

实现 

1. 随机生成一个遍历的轮询顺序 `pollOrder` 并根据 Channel 地址生成锁定顺序 `lockOrder`；
2. 根据 `pollOrder` 遍历所有的 `case` 查看是否有可以立刻处理的 Channel；
	- 如果存在就直接获取 `case` 对应的索引并返回；
	- 如果不存在就会创建 [`runtime.sudog`](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fblob%2Fcfe3cd903f018dec3cb5997d53b1744df4e53909%2Fsrc%2Fruntime%2Fruntime2.go%23L342-L368) 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 [`runtime.gopark`](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fblob%2Fc112289ee4141ebc31db50328c355b01278b987b%2Fsrc%2Fruntime%2Fproc.go%23L287-L305) 挂起当前 Goroutine 等待调度器的唤醒；
3. 当调度器唤醒当前 Goroutine 时就会再次按照 `lockOrder` 遍历所有的 `case`，从中查找需要被处理的 [`runtime.sudog`](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fblob%2Fcfe3cd903f018dec3cb5997d53b1744df4e53909%2Fsrc%2Fruntime%2Fruntime2.go%23L342-L368) 结构对应的索引；

```go
// compiler implements
//
// select {
// case v = <-c:
//    ... foo
// default:
//    ... bar
// }
//
// as
//
// if selectnbrecv(&v, c) {
//    ... foo
// } else {
//    ... bar
// }
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
   selected, _ = chanrecv(c, elem, false)  //非阻塞
   return
}
```

## 死锁（deadlock）

指两个或两个以上的协程的执行过程中，由于竞争资源或由于彼此通信而造成的一种阻塞的现象。

在非缓冲信道若发生只流入不流出，或只流出不流入，就会发生死锁。

下面是一些死锁的例子

1、

```
package main

func main() {
   ch := make(chan int)
   ch <- 3
}
```

上面情况，向非缓冲通道写数据会发生阻塞，导致死锁。解决办法创建缓冲区 ch := make(chan int，3)

2、

```
package main

import (
   "fmt"
)

func main() {
   ch := make(chan int)
   fmt.Println(<-ch)
}
```

向非缓冲通道读取数据会发生阻塞，导致死锁。 解决办法开启缓冲区，先向channel写入数据。

3、

```
package main

func main() {
   ch := make(chan int, 3)
   ch <- 3
   ch <- 4
   ch <- 5
   ch <- 6
}
```

写入数据超过缓冲区数量也会发生死锁。解决办法将写入数据取走。

死锁的情况有很多这里不再赘述。
还有一种情况，向关闭的channel写入数据，不会产生死锁，产生panic。

```
package main

func main() {
    ch := make(chan int, 3)
    ch <- 1
    close(ch)
    ch <- 2
}
```

解决办法别向关闭的channel写入数据。