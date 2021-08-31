# Channel

go 的有锁数据结构，CSP 概念的组成因子之一。

## basic usage

阻塞式 channel ：

```go
var a = make(chan int)
```

非阻塞 channel：

```go
var a = make(chan int, 10)
```

阻塞和非阻塞关键就在是否有 capacity。没有 capacity 的话，channel 也就只是个同步通信工具。

向 channel 中发送内容：

```go
ch := make(chan int, 100)
ch <- 1
```

从 channel 中接收内容：

```go
var i = <- ch
```

关闭 channel：

```go
close(ch)
```

注意，已关闭的 channel，再次关闭会 panic

```go
close(ch)
close(ch) // panic: close of closed channel
```

在 channel 关闭时自动退出循环

```go
func main() {
    ch := make(chan int, 100)
    for elem := range ch { // 主要就是这里的 for range...
        fmt.Println(i)
    }
}
```

获取 channel 中元素数量、buffer 容量：

```go
func main() {
    ch := make(chan int, 100)
    ch <- 1
    fmt.Println(len(ch)) // 1
    fmt.Println(cap(ch)) // 100
}
```

注意，len 和 cap 并不是函数调用。编译后是直接去取 hchan 的 field 了。

## closed channel

被关闭的 channel 不能再向其中发送内容，否则会 panic

```go
ch := make(chan int)
close(ch)
ch <- 1 // panic: send on closed channel
```

注意，如果 close channel 时，有 sender goroutine 挂在 channel 的阻塞发送队列中，会导致 panic：

```go
func main() {
    ch := make(chan int)
    go func() { ch <- 1 }() // panic: send on closed channel
    time.Sleep(time.Second)
    go func() { close(ch) }()
    time.Sleep(time.Second)
    x, ok := <-ch
    fmt.Println(x, ok)
}
```

close 一个 channel 会唤醒所有等待在该 channel 上的 g，并使其进入 Grunnable 状态，这时这些 writer goroutine 会发现该 channel 已经是 closed 状态，就 panic了。

在不确定是否还有 goroutine 需要向 channel 发送数据时，请勿贸然关闭 channel。

可以从已经 closed 的 channel 中接收值：

```go
ch := make(chan int)
close(ch)
x := <-ch
```

如果 channel 中有值，这里特指带 buffer 的 channel，那么就从 channel 中取，如果没有值，那么会返回 channel 元素的 0 值。

区分是返回的零值还是 buffer 中的值可使用 comma, ok 语法：

```go
x, ok := <-ch
```

若 ok 为 false，表明 channel 已被关闭，所得的是无效的值。

## nil channel

不进行初始化，即不调用 make 来赋值的 channel 称为 nil channel：

```go
var a chan int
```

关闭一个 nil channel 会直接 panic

```go
var a chan int
close(a) // panic: close of nil channel
```

## close principle

一个 sender，多个 receiver，由 sender 来关闭 channel，通知数据已发送完毕。

一旦 sender 有多个，可能就无法判断数据是否完毕了。这时候可以借助外部额外 channel 来做信号广播。这种做法类似于 done channel，或者 stop channel。

可参考：[graceful close channel](https://go101.org/article/channel-closing.html)

如果确定不会有 goroutine 在通信过程中被阻塞，也可以不关闭 channel，等待 GC 对其进行回收。

# 源码分析

## hchan

hchan 是 channel 在 runtime 中的数据结构

```go
// channel 在 runtime 中的结构体
type hchan struct {
    // 队列中目前的元素计数
    qcount uint // total data in the queue
    // 环形队列的总大小，ch := make(chan int, 10) => 就是这里这个 10
    dataqsiz uint // size of the circular queue
    // void * 的内存 buffer 区域
    buf unsafe.Pointer // points to an array of dataqsiz elements
    // sizeof chan 中的数据
    elemsize uint16
    // 是否已被关闭
    closed uint32
    // runtime._type，代表 channel 中的元素类型的 runtime 结构体
    elemtype *_type // element type
    // 发送索引
    sendx uint // send index
    // 接收索引
    recvx uint // receive index
    // 接收 goroutine 对应的 sudog 队列
    recvq waitq // list of recv waiters
    // 发送 goroutine 对应的 sudog 队列
    sendq waitq // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

## init

```go
// 初始化 channel
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }

    if size < 0 || uintptr(size) > maxSliceCap(elem.size) || uintptr(size)*elem.size > _MaxMem-hchanSize {
        panic(plainError("makechan: size out of range"))
    }

    // Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // 如果 hchan 中的元素不包含有指针，那么就没什么和 GC 相关的信息了
    var c *hchan
    // 可以学习一下这种空 switch 的写法，比 if else 好看一些
    switch {
    case size == 0 || elem.size == 0:
        // 如果 channel 的缓冲区大小是 0: var a = make(chan int)
        // 或者 channel 中的元素大小是 0: struct{}{}
        // Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = unsafe.Pointer(c)
    case elem.kind&kindNoPointers != 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        // 通过位运算知道 channel 中的元素不包含指针
        // 占用的空间比较容易计算
        // 直接用 元素数*元素大小 + channel 必须的空间就行了
        // 这种情况下 gc 不会对 channel 中的元素进行 scan
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Elements contain pointers.
        // 和上面那个 case 的写法的区别:调用了两次分配空间的函数 new/mallocgc
        c = new(hchan)
        c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    return c
}
```

## send

```go
// entry point for c <- x from compiled code
// 英文写的比较明白了。。
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 应用层的 channel 为空
    // 例如 var a chan int
    // a<-1
    if c == nil {
        if !block {
            return false
        }
        // nil channel 发送数据会永远阻塞下去
        // PS：注意，会发生 panic 那种情况是 channel 被 closed 了，不是 nil channel
        // 挂起当前 goroutine
        gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not closed, we observe that the channel is
    // not ready for sending. Each of these observations is a single word-sized read
    // (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
    // Because a closed channel cannot transition from 'ready for sending' to
    // 'not ready for sending', even if the channel is closed between the two observations,
    // they imply a moment between the two when the channel was both not yet closed
    // and not ready for sending. We behave as if we observed the channel at that moment,
    // and report that the send cannot proceed.
    //
    // It is okay if the reads are reordered here: if we observe that the channel is not
    // ready for sending and then observe that it is not closed, that implies that the
    // channel wasn't closed during the first observation.
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    // channel 已被关闭，panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        // 寻找一个等待中的 receiver
        // 越过 channel 的 buffer
        // 直接把要发的数据拷贝给这个 receiver
        // 然后就返
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // qcount 是 buffer 中已塞进的元素数量
    // dataqsize 是 buffer 的总大小
    // 说明还有余量
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)

        // 将 goroutine 的数据拷贝到 buffer 中
        typedmemmove(c.elemtype, qp, ep)

        // 将发送 index 加一
        c.sendx++
        // 环形队列，所以如果已经加到最大了，就回 0
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        // 将 buffer 的元素计数 +1
        c.qcount++
        unlock(&c.lock)
        return true
    }

    if !block {
        unlock(&c.lock)
        return false
    }

    // Block on the channel. Some receiver will complete our operation for us.
    // 在 channel 上阻塞，receiver 会帮我们完成后续的工作
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    // 打包 sudog
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil

    // 将当前这个发送 goroutine 打包后的 sudog 入队到 channel 的 sendq 队列中
    c.sendq.enqueue(mysg)

    // 将这个发送 g 从 Grunning -> Gwaiting
    // 进入休眠
    goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

    // someone woke us up.
    // 这里是被唤醒后要执行的代码
    if mysg != gp.waiting {
        // 先判断当前是不是合法的休眠中
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        // 唤醒后发现 channel 被人关了，气啊
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

// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
// 英文已经说的比较明白了。。
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // receiver 的 sudog 已经在对应区域分配过空间
    // 我们只要把数据拷贝过去
    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep)
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }

    // Gwaiting -> Grunnable
    goready(gp, skip+1)
}

```

## receive

```go
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}


func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    // 如果在 nil channel 上进行 recv 操作，那么会永远阻塞
    if c == nil {
        if !block {
            // 非阻塞的情况下
            // 要直接返回
            // 非阻塞出现在一些 select 的场景中
            // 参见 selectnbrecv/selectnbrecv2
            return
        }
        // 当前 goroutine: Grunning -> Gwaiting
        // 其实就是该 goroutine 直接泄露 leak 了
        gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)

        // 放个 throw 有点莫名其妙
        // 不过这段代码确实永远达到不了
        throw("unreachable")
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not ready for receiving, we observe that the
    // channel is not closed. Each of these observations is a single word-sized read
    // (first c.sendq.first or c.qcount, and second c.closed).
    // Because a channel cannot be reopened, the later observation of the channel
    // being not closed implies that it was also not closed at the moment of the
    // first observation. We behave as if we observed the channel at that moment
    // and report that the receive cannot proceed.
    //
    // The order of operations is important here: reversing the operations can lead to
    // incorrect behavior when racing with a close.
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        // 非阻塞且没内容可收的情况下要直接返回
        // 两个 bool 的零值就是 false，false
        return
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    // 当前 channel 中没有数据可读
    // 直接返回 not selected
    if c.closed != 0 && c.qcount == 0 {
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }

    // sender 队列中有 sudog 在等待
    // 直接从该 sudog 中获取数据拷贝到当前 g 即可
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)

        // 直接从 buffer 里拷贝数据
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        // 接收索引 +1
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        // buffer 元素计数 -1
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    if !block {
        unlock(&c.lock)
        // 非阻塞时，且无数据可收
        // 始终不选中，这是在 buffer 中没内容的时候
        return false, false
    }

    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    // 打包成 sudog
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    // 进入 recvq 队列
    c.recvq.enqueue(mysg)

    // Grunning -> Gwaiting
    goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

    // someone woke us up
    // 被唤醒
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    // 如果 channel 未被关闭，那就是真的 recv 到数据了
    return true, !closed
}

// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
        if ep != nil {
            // copy data from sender
            recvDirect(c.elemtype, sg, ep)
        }
    } else {
        // Queue is full. Take the item at the
        // head of the queue. Make the sender enqueue
        // its item at the tail of the queue. Since the
        // queue is full, those are both the same slot.
        qp := chanbuf(c, c.recvx)

        // copy data from queue to receiver
        // 英文写的很明白
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        // copy data from sender to queue
        // 英文写的很明白
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }

    // Gwaiting -> Grunnable
    goready(gp, skip+1)
}
```

## close

```go
func closechan(c *hchan) {
    // 关闭一个 nil channel 会直接 panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    // 上锁，这个锁的粒度比较大，一直到释放完所有的 sudog 才解锁
    lock(&c.lock)

    // 在 close channel 时，如果 channel 已经关闭过了
    // 直接触发 panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    c.closed = 1

    var glist *g

    // release all readers
    for {
        sg := c.recvq.dequeue()
        // 弹出的 sudog 是 nil
        // 说明读队列已经空了
        if sg == nil {
            break
        }

        // sg.elem unsafe.Pointer，指向 sudog 的数据元素
        // 该元素可能在堆上分配，也可能在栈上
        if sg.elem != nil {
            // 释放对应的内存
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }

        // 将 goroutine 入 glist
        // 为最后将全部 goroutine 都 ready 做准备
        gp := sg.g
        gp.param = nil
        gp.schedlink.set(glist)
        glist = gp
    }

    // release all writers (they will panic)
    // 将所有挂在 channel 上的 writer 从 sendq 中弹出
    // 该操作会使所有 writer panic
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }

        // 将 goroutine 入 glist
        // 为最后将全部 goroutine 都 ready 做准备
        gp := sg.g
        gp.param = nil
        gp.schedlink.set(glist)
        glist = gp
    }

    // 在释放所有挂在 channel 上的读或写 sudog 时
    // 是一直在临界区的
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for glist != nil {
        gp := glist
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        // 使 g 的状态切换到 Grunnable
        goready(gp, 3)
    }
}
```

## 一些问题

Q: 如果有多个channel同时唤醒同一个goroutine，这个并发控制是怎么做的？

Q: 为什么向 channel 发数据的时候，会直接把数据从一个 goroutine 的栈拷贝到另一个 goroutine 的栈？


<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
