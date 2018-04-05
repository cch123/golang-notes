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
var i := <- ch
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
虽然看起来是 ch <- 1 导致的 panic，但实际上 close 才是罪魁祸首，在不确定是否还有 goroutine 需要向 channel 发送数据时，请勿贸然关闭 channel。

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
var a = chan int
```
关闭一个 nil channel 会直接 panic
```go
var a = chan int
close(a) // panic: close of nil channel
```

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

## receive

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

    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(unsafe.Pointer(c), callerpc, funcPC(closechan))
        racerelease(unsafe.Pointer(c))
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
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
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
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
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


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTY2OTk4NTMzMywxMzc4NzUyODgzXX0=
-->
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTI2NzI2MjYxN119
-->