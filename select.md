# Select

select 的实现在目前 Go 的 master 分支已经有所修改，所以本文只针对 1.10 之前的版本。

## 基本用法

非阻塞 select:

```go
select {
    case <-ch:
    default:
}
```

当 ch 中无数据时，不会阻塞，会直接走 default 分支。这种特性可以用来实现 trylock。

对 nil 进行发送或者接收都会永远阻塞，被包在 select 里也不例外:

```go
var a chan int
select {
    case <-ch:
}

println("can not be printed")
```

for select 组合起来用很常见，不过需要注意，下面的两种场景可能会造成问题:

```go
for {
    select {
        case d := <-ch:
        default:
    }
}
```

这种写法比较危险，如果 ch 中没有数据就会一直死循环 cpu 爆炸。

```go
for {
    select {
        case d := <-ch:
    }
}
```

你以为这么写就没事了？啊当然，一般情况下还好。但如果 ch 被其它 goroutine close 掉了，那么 d:= <-ch 这种形式就是永远不阻塞，并且会一直返回零值了。如果不想关注这种情况，并且 select 中实际只操作一个 channel，建议写成 for range 形式:

```go
for d := range ch {
    // do some happy things with d
}
```

这样 ch 关闭的时候 for 循环会自动退出。

如果 select 中需要监听多个 channel，并且这些 channel 可能被关闭，那需要老老实实地写双返回值的 channel 取值表达式:

```go
outer:
for {
    select {
        case d, ok := <-ch1:
            if !ok {
                break outer
            }
        case d, ok := <-ch2:
            if !ok {
                break outer
            }
    }
}
```

当然，如果你不确定，可以用下面的 demo 进行验证:

```go
package main

import "time"

func main() {
    var ch1 chan int
    var ch2 = make(chan int)
    close(ch2)
    go func() {
        for {
            select {
            case d := <-ch1:
                println("ch1", d)
            case d := <-ch2:
                println("ch2", d)
            }
        }
    }()
    time.Sleep(time.Hour)
}
```

尽管 ch2 已经被关闭，依然会不断地进入 case d:= <-ch2 中。因此在使用 for select 做设计时，请务必考虑当监听的 channel 在外部被正常或意外关闭后会有什么样的后果。

## 源码分析

### 数据结构

```go
const (
    // scase.kind
    // send 或者 recv 发生在一个 nil channel 上，就有可能出现这种情况
    caseNil = iota
    caseRecv
    caseSend
    caseDefault
)

// Select statement header.
// Known to compiler.
// Changes here must also be made in src/cmd/internal/gc/select.go's selecttype.
// select 的本体数据结构
type hselect struct {
    // 总 case 数
    tcase uint16 // total count of scase[]
    // 当前已初始化填充的 case
    ncase uint16 // currently filled scase[]
    // 轮询顺序
    pollorder *uint16 // case poll order
    // case 的 channel 的加锁顺序
    lockorder *uint16 // channel lock order
    // case 数组，为了节省一个指针的 8 个字节搞成这样的结构
    // 实际上要访问后面的值，还是需要进行指针移动
    // 指针移动使用 runtime 内部的 add 函数
    scase [1]scase // one per case (in order of appearance)
}

// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/internal/gc/select.go's selecttype.
// select 中每一个 case 的数据结构定义
type scase struct {
    // 数据元素
    elem unsafe.Pointer // data element
    // channel 本体
    c *hchan // chan
    // 这个是指 ip 寄存器么？
    pc uintptr // return pc (for race detector / msan)
    // 应该是 reflect.Kind
    kind uint16
    // _,ok:= <- ch 里的 ok 吧
    receivedp *bool // pointer to received bool, if any

    // TODO 这个是干啥的
    releasetime int64
}

var (
    chansendpc = funcPC(chansend)
    chanrecvpc = funcPC(chanrecv)
)

func selectsize(size uintptr) uintptr {
    selsize := unsafe.Sizeof(hselect{}) +
        (size-1)*unsafe.Sizeof(hselect{}.scase[0]) +
        size*unsafe.Sizeof(*hselect{}.lockorder) +
        size*unsafe.Sizeof(*hselect{}.pollorder)
    return round(selsize, sys.Int64Align)
}

```

### select 初始化

```go
// 创建一个 select 结构体
// 每次写出：
// select { => 在这行会调用 newselect
// 可以使用 go tool compile -S 来查看
// ...
// }
func newselect(sel *hselect, selsize int64, size int32) {
    if selsize != int64(selectsize(uintptr(size))) {
        print("runtime: bad select size ", selsize, ", want ", selectsize(uintptr(size)), "\n")
        throw("bad select size")
    }
    sel.tcase = uint16(size)
    // 初始已填充的 case 数自然是 0
    sel.ncase = 0
    // hselect 这个结构体的是长下面这样：
    // header |scase[0]|scase[1]|scase[2]|lockorder[0]|lockorder[1]|lockorder[2]|pollorder[0]|pollorder[1]|pollorder[2]
    // 各个 channel 加锁的顺序
    sel.lockorder = (*uint16)(add(unsafe.Pointer(&sel.scase), uintptr(size)*unsafe.Sizeof(hselect{}.scase[0])))
    // case 轮询的顺序初始化
    sel.pollorder = (*uint16)(add(unsafe.Pointer(sel.lockorder), uintptr(size)*unsafe.Sizeof(*hselect{}.lockorder)))

}

```

### select send

```go
// select {
//   case ch<-1: ==> 这时候就会调用 selectsend
// }
func selectsend(sel *hselect, c *hchan, elem unsafe.Pointer) {
    pc := getcallerpc()
    i := sel.ncase
    if i >= sel.tcase {
        throw("selectsend: too many cases")
    }
    sel.ncase = i + 1
    if c == nil {
        return
    }

    // 初始化对应的 scase 结构
    cas := (*scase)(add(unsafe.Pointer(&sel.scase), uintptr(i)*unsafe.Sizeof(sel.scase[0])))
    cas.pc = pc
    cas.c = c
    cas.kind = caseSend
    cas.elem = elem

}
```

```go
// compiler implements
//
//    select {
//    case c <- v:
//        ... foo
//    default:
//        ... bar
//    }
//
// as
//
//    if selectnbsend(c, v) {
//        ... foo
//    } else {
//        ... bar
//    }
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc())
}
```

### select receive

```go
// select {
// case <-ch: ==> 这时候就会调用 selectrecv
// case ,ok <- ch: 也可以这样写
// 在 ch 被关闭时，这个 case 每次都可能被轮询到
// }
func selectrecv(sel *hselect, c *hchan, elem unsafe.Pointer, received *bool) {
    pc := getcallerpc()
    i := sel.ncase
    if i >= sel.tcase {
        throw("selectrecv: too many cases")
    }
    sel.ncase = i + 1
    if c == nil {
        return
    }

    // 初始化对应的 scase 结构
    cas := (*scase)(add(unsafe.Pointer(&sel.scase), uintptr(i)*unsafe.Sizeof(sel.scase[0])))
    cas.pc = pc
    cas.c = c
    cas.kind = caseRecv
    cas.elem = elem
    // _, ok => 这里的 ok
    cas.receivedp = received

}

```

```go
// compiler implements
//
//    select {
//    case v = <-c:
//        ... foo
//    default:
//        ... bar
//    }
//
// as
//
//    if selectnbrecv(&v, c) {
//        ... foo
//    } else {
//        ... bar
//    }
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
    selected, _ = chanrecv(c, elem, false)
    return
}
```

```go
// compiler implements
//
//    select {
//    case v, ok = <-c:
//        ... foo
//    default:
//        ... bar
//    }
//
// as
//
//    if c != nil && selectnbrecv2(&v, &ok, c) {
//        ... foo
//    } else {
//        ... bar
//    }
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
    // TODO(khr): just return 2 values from this function, now that it is in Go.
    selected, *received = chanrecv(c, elem, false)
    return
}
```

### select default

```go
// select {
//   default: ==> 这时候就会调用 selectdefault
// }
func selectdefault(sel *hselect) {
    pc := getcallerpc()
    i := sel.ncase
    if i >= sel.tcase {
        throw("selectdefault: too many cases")
    }
    sel.ncase = i + 1

    // 初始化对应的 scase 结构
    cas := (*scase)(add(unsafe.Pointer(&sel.scase), uintptr(i)*unsafe.Sizeof(sel.scase[0])))
    cas.pc = pc
    cas.c = nil
    cas.kind = caseDefault

}

```

### select 执行

加解锁:

```go
// 对所有 case 对应的 channel 加锁
// 需要按照 lockorder 数组中的元素索引来搞
// 否则可能有循环等待的死锁
func sellock(scases []scase, lockorder []uint16) {
    var c *hchan
    for _, o := range lockorder {
        c0 := scases[o].c
        if c0 != nil && c0 != c {
            c = c0
            lock(&c.lock)
        }
    }
}

// 解锁，比较简单
func selunlock(scases []scase, lockorder []uint16) {
    // We must be very careful here to not touch sel after we have unlocked
    // the last lock, because sel can be freed right after the last unlock.
    // Consider the following situation.
    // First M calls runtime·park() in runtime·selectgo() passing the sel.
    // Once runtime·park() has unlocked the last lock, another M makes
    // the G that calls select runnable again and schedules it for execution.
    // When the G runs on another M, it locks all the locks and frees sel.
    // Now if the first M touches sel, it will access freed memory.
    for i := len(scases) - 1; i >= 0; i-- {
        c := scases[lockorder[i]].c
        if c == nil {
            break
        }
        if i > 0 && c == scases[lockorder[i-1]].c {
            continue // will unlock it on the next iteration
        }
        unlock(&c.lock)
    }
}

```

```go
func selparkcommit(gp *g, _ unsafe.Pointer) bool {
    // This must not access gp's stack (see gopark). In
    // particular, it must not access the *hselect. That's okay,
    // because by the time this is called, gp.waiting has all
    // channels in lock order.
    var lastc *hchan
    for sg := gp.waiting; sg != nil; sg = sg.waitlink {
        if sg.c != lastc && lastc != nil {
            // As soon as we unlock the channel, fields in
            // any sudog with that channel may change,
            // including c and waitlink. Since multiple
            // sudogs may have the same channel, we unlock
            // only after we've passed the last instance
            // of a channel.
            unlock(&lastc.lock)
        }
        lastc = sg.c
    }
    if lastc != nil {
        unlock(&lastc.lock)
    }
    return true
}

// 没有 case 直接 block，这种情况下让该 goroutine 挂起进休眠
func block() {
    gopark(nil, nil, "select (no cases)", traceEvGoStop, 1) // forever
}

```

select 执行:

```go

// selectgo implements the select statement.
//
// *sel is on the current goroutine's stack (regardless of any
// escaping in selectgo).
//
// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// selectgo 是在初始化完成之后执行 select 逻辑的函数
// 返回值是要执行的 scase 的 index
func selectgo(sel *hselect) int {
    if sel.ncase != sel.tcase {
        throw("selectgo: case count mismatch")
    }

    // len 和 cap 就是 scase 的总数
    // 这时候 ncase 和 tcase 已经是相等的了
    scaseslice := slice{unsafe.Pointer(&sel.scase), int(sel.ncase), int(sel.ncase)}
    scases := *(*[]scase)(unsafe.Pointer(&scaseslice))

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
        for i := 0; i < int(sel.ncase); i++ {
            scases[i].releasetime = -1
        }
    }

    // The compiler rewrites selects that statically have
    // only 0 or 1 cases plus default into simpler constructs.
    // The only way we can end up with such small sel.ncase
    // values here is for a larger select in which most channels
    // have been nilled out. The general code handles those
    // cases correctly, and they are rare enough not to bother
    // optimizing (and needing to test).

    // generate permuted order
    pollslice := slice{unsafe.Pointer(sel.pollorder), int(sel.ncase), int(sel.ncase)}
    pollorder := *(*[]uint16)(unsafe.Pointer(&pollslice))

    // 这个洗牌有点挫。。。。
    for i := 1; i < int(sel.ncase); i++ {
        j := fastrandn(uint32(i + 1))
        // 这时候 pollorder[i] 其实就是 i
        // 这么写少了一次交换
        pollorder[i] = pollorder[j]
        pollorder[j] = uint16(i)
    }

    // sort the cases by Hchan address to get the locking order.
    // simple heap sort, to guarantee n log n time and constant stack footprint.
    // 按 hchan 的地址来进行排序，以生成加锁顺序
    // 用堆排序来保证 nLog(n) 的时间复杂度
    lockslice := slice{unsafe.Pointer(sel.lockorder), int(sel.ncase), int(sel.ncase)}
    lockorder := *(*[]uint16)(unsafe.Pointer(&lockslice))
    for i := 0; i < int(sel.ncase); i++ {
        // 初始化 lockorder 数组
        j := i
        // Start with the pollorder to permute cases on the same channel.
        c := scases[pollorder[i]].c
        for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
            k := (j - 1) / 2
            lockorder[j] = lockorder[k]
            j = k
        }
        lockorder[j] = pollorder[i]
    }
    for i := int(sel.ncase) - 1; i >= 0; i-- {
        // 堆排序
        o := lockorder[i]
        c := scases[o].c
        lockorder[i] = lockorder[0]
        j := 0
        for {
            k := j*2 + 1
            if k >= i {
                break
            }
            if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
                k++
            }
            if c.sortkey() < scases[lockorder[k]].c.sortkey() {
                lockorder[j] = lockorder[k]
                j = k
                continue
            }
            break
        }
        lockorder[j] = o
    }
    /*
        for i := 0; i+1 < int(sel.ncase); i++ {
            if scases[lockorder[i]].c.sortkey() > scases[lockorder[i+1]].c.sortkey() {
                print("i=", i, " x=", lockorder[i], " y=", lockorder[i+1], "\n")
                throw("select: broken sort")
            }
        }
    */

    // lock all the channels involved in the select
    // 对涉及到的所有 channel 都加锁
    sellock(scases, lockorder)

    var (
        gp     *g
        sg     *sudog
        c      *hchan
        k      *scase
        sglist *sudog
        sgnext *sudog
        qp     unsafe.Pointer
        nextp  **sudog
    )

loop:
    // pass 1 - look for something already waiting
    var dfli int
    var dfl *scase
    var casi int
    var cas *scase

    // 虽然看着是一个 for 循环
    // 但实际上有 case ready 的时候
    // 直接就用 goto 跳出了
    for i := 0; i < int(sel.ncase); i++ {
        // 按 pollorder 的顺序进行遍历
        casi = int(pollorder[i])
        cas = &scases[casi]
        c = cas.c

        switch cas.kind {
        case caseNil:
            /*
             * var nil_chan chan int
             * var non_nil_chan chan int = make(chan int)
             * select {
             *   case <-nil_chan:
             *        // here
             *   case <-non_nil_chan:
             * }
             */
            continue

        case caseRecv:
            // <- ch 的情况
            // 根据有没有等待的 goroutine 队列执行不同的操作
            sg = c.sendq.dequeue()
            if sg != nil {
                goto recv
            }
            if c.qcount > 0 {
                goto bufrecv
            }
            if c.closed != 0 {
                goto rclose
            }

        case caseSend:
            // ch <- 1 的情况，也是一些基本的 channel 操作
            if c.closed != 0 {
                goto sclose
            }
            sg = c.recvq.dequeue()
            if sg != nil {
                goto send
            }
            if c.qcount < c.dataqsiz {
                goto bufsend
            }

        case caseDefault:
            dfli = casi
            dfl = cas
        }
    }

    // 这里是在前面进了 caseDefault 才会走到
    // 感觉这代码写的也挺挫的
    if dfl != nil {
        // 之间是进了 default，才会进这个 if
        selunlock(scases, lockorder)
        casi = dfli
        cas = dfl
        goto retc
    }

    // pass 2 - enqueue on all chans
    // 没有任何一个 case 满足，且没有 default
    // 这种情况下需要把当前的 goroutine 入队所有 channel 的等待队列里
    gp = getg()
    if gp.waiting != nil {
        throw("gp.waiting != nil")
    }
    nextp = &gp.waiting
    // 按照加锁的顺序把 gorutine 入每一个 channel 的等待队列
    for _, casei := range lockorder {
        casi = int(casei)
        cas = &scases[casi]
        // 空 channel
        if cas.kind == caseNil {
            continue
        }
        c = cas.c
        sg := acquireSudog()
        sg.g = gp
        sg.isSelect = true
        // No stack splits between assigning elem and enqueuing
        // sg on gp.waiting where copystack can find it.
        sg.elem = cas.elem
        sg.releasetime = 0
        if t0 != 0 {
            sg.releasetime = -1
        }
        sg.c = c
        // Construct waiting list in lock order.
        *nextp = sg
        nextp = &sg.waitlink

        switch cas.kind {
        case caseRecv:
            // recv 的情况进 recvq
            c.recvq.enqueue(sg)

        case caseSend:
            // send 的情况进 sendq
            c.sendq.enqueue(sg)
        }
    }

    // wait for someone to wake us up
    gp.param = nil
    // 当前 goroutine 进入休眠，等待被唤醒
    gopark(selparkcommit, nil, "select", traceEvGoBlockSelect, 1)

    sellock(scases, lockorder)

    gp.selectDone = 0
    sg = (*sudog)(gp.param)
    gp.param = nil

    // pass 3 - dequeue from unsuccessful chans
    // otherwise they stack up on quiet channels
    // record the successful case, if any.
    // We singly-linked up the SudoGs in lock order.
    // 被唤醒后执行下面的代码
    casi = -1
    cas = nil
    sglist = gp.waiting
    // Clear all elem before unlinking from gp.waiting.
    for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
        sg1.isSelect = false
        sg1.elem = nil
        sg1.c = nil
    }
    gp.waiting = nil

    for _, casei := range lockorder {
        k = &scases[casei]
        if k.kind == caseNil {
            continue
        }
        if sglist.releasetime > 0 {
            k.releasetime = sglist.releasetime
        }
        if sg == sglist {
            // sg has already been dequeued by the G that woke us up.
            casi = int(casei)
            cas = k
        } else {
            c = k.c
            if k.kind == caseSend {
                c.sendq.dequeueSudoG(sglist)
            } else {
                c.recvq.dequeueSudoG(sglist)
            }
        }
        sgnext = sglist.waitlink
        sglist.waitlink = nil
        releaseSudog(sglist)
        sglist = sgnext
    }

    if cas == nil {
        // We can wake up with gp.param == nil (so cas == nil)
        // when a channel involved in the select has been closed.
        // It is easiest to loop and re-run the operation;
        // we'll see that it's now closed.
        // Maybe some day we can signal the close explicitly,
        // but we'd have to distinguish close-on-reader from close-on-writer.
        // It's easiest not to duplicate the code and just recheck above.
        // We know that something closed, and things never un-close,
        // so we won't block again.
        goto loop
    }

    c = cas.c

    if cas.kind == caseRecv && cas.receivedp != nil {
        *cas.receivedp = true
    }

    if msanenabled {
        if cas.kind == caseRecv && cas.elem != nil {
            msanwrite(cas.elem, c.elemtype.size)
        } else if cas.kind == caseSend {
            msanread(cas.elem, c.elemtype.size)
        }
    }

    selunlock(scases, lockorder)
    goto retc

bufrecv:
    // can receive from buffer
    if msanenabled && cas.elem != nil {
        msanwrite(cas.elem, c.elemtype.size)
    }
    if cas.receivedp != nil {
        *cas.receivedp = true
    }
    qp = chanbuf(c, c.recvx)
    if cas.elem != nil {
        typedmemmove(c.elemtype, cas.elem, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    selunlock(scases, lockorder)
    goto retc

bufsend:
    // can send to buffer
    if msanenabled {
        msanread(cas.elem, c.elemtype.size)
    }
    typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    selunlock(scases, lockorder)
    goto retc

recv:
    // can receive from sleeping sender (sg)
    recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    if cas.receivedp != nil {
        *cas.receivedp = true
    }
    goto retc

rclose:
    // read at end of closed channel
    selunlock(scases, lockorder)
    if cas.receivedp != nil {
        *cas.receivedp = false
    }
    if cas.elem != nil {
        typedmemclr(c.elemtype, cas.elem)
    }
    goto retc

send:
    // can send to a sleeping receiver (sg)
    if msanenabled {
        msanread(cas.elem, c.elemtype.size)
    }
    send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    goto retc

retc:
    if cas.releasetime > 0 {
        blockevent(cas.releasetime-t0, 1)
    }
    return casi

sclose:
    // send on closed channel
    // 向关闭 channel 发送数据
    // 直接 panic
    selunlock(scases, lockorder)
    panic(plainError("send on closed channel"))
}

```

## 一些问题

Q: 如果select多个channel，有一个channel触发了，其他channel的waitlist需要不要主动去除？还是一直在那等着？

A: waitlist 的出列是由 `func (q *waitq) dequeue() *sudog` 函数控制的，每个 sudog 携带了一个 `selectDone` 标志位，通过 `cas` 操作在每次 `dequeue` 的时候「惰性」去除队列中无效的元素

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
