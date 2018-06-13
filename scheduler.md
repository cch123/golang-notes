# 调度

## 基本数据结构

goroutine 在 runtime 中的数据结构:

```go
// stack 描述的是 Go 的执行栈，下界和上界分别为 [lo, hi]
// 如果从传统内存布局的角度来讲，Go 的栈实际上是分配在 C 语言中的堆区的
// 所以才能比 ulimit -s 的 stack size 还要大(1GB)
type stack struct {
    lo uintptr
    hi uintptr
}

// g 的运行现场
type gobuf struct {
    sp   uintptr    // sp 寄存器
    pc   uintptr    // pc 寄存器
    g    guintptr   // g 指针
    ctxt unsafe.Pointer // 这个似乎是用来辅助 gc 的
    ret  sys.Uintreg
    lr   uintptr    // 这是在 arm 上用的寄存器，不用关心
    bp   uintptr    // 开启 GOEXPERIMENT=framepointer，才会有这个
}


type g struct {
    // 简单数据结构，lo 和 hi 成员描述了栈的下界和上界内存地址
    stack       stack
    // 在函数的栈增长 prologue 中用 sp 寄存器和 stackguard0 来做比较
    // 如果 sp 比 stackguard0 小(因为栈向低地址方向增长)，那么就触发栈拷贝和调度
    // 正常情况下 stackguard0 = stack.lo + StackGuard
    // 不过 stackguard0 在需要进行调度时，会被修改为 StackPreempt
    // 以触发抢占s
    stackguard0 uintptr
    // stackguard1 是在 C 栈增长 prologue 作对比的对象
    // 在 g0 和 gsignal 栈上，其值为 stack.lo+StackGuard
    // 在其它的栈上这个值是 ~0(按 0 取反)以触发 morestack 调用(并 crash)
    stackguard1 uintptr

    _panic         *_panic
    _defer         *_defer
    m              *m             // 当前与 g 绑定的 m
    sched          gobuf          // goroutine 的现场
    syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
    syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
    stktopsp       uintptr        // expected sp at top of stack, to check in traceback
    param          unsafe.Pointer // wakeup 时的传入参数
    atomicstatus   uint32
    stackLock      uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
    goid           int64  // goroutine id
    waitsince      int64  // g 被阻塞之后的近似时间
    waitreason     string // if status==Gwaiting
    schedlink      guintptr
    preempt        bool     // 抢占标记，这个为 true 时，stackguard0 是等于 stackpreempt 的
    throwsplit     bool     // must not split stack
    raceignore     int8     // ignore race detection events
    sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
    sysexitticks   int64    // syscall 返回之后的 cputicks，用来做 tracing
    traceseq       uint64   // trace event sequencer
    tracelastp     puintptr // last P emitted an event for this goroutine
    lockedm        muintptr // 如果调用了 LockOsThread，那么这个 g 会绑定到某个 m 上
    sig            uint32
    writebuf       []byte
    sigcode0       uintptr
    sigcode1       uintptr
    sigpc          uintptr
    gopc           uintptr // 创建该 goroutine 的语句的指令地址
    startpc        uintptr // goroutine 函数的指令地址
    racectx        uintptr
    waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
    cgoCtxt        []uintptr      // cgo traceback context
    labels         unsafe.Pointer // profiler labels
    timer          *timer         // time.Sleep 缓存的定时器
    selectDone     uint32         // 该 g 是否正在参与 select，是否已经有人从 select 中胜出
}
```

当 g 遇到阻塞，或需要等待的场景时，会被打包成 sudog 这样一个结构。一个 g 可能被打包为多个 sudog 分别挂在不同的等待队列上:

```go
// sudog 代表在等待列表里的 g，比如向 channel 发送/接收内容时
// 之所以需要 sudog 是因为 g 和同步对象之间的关系是多对多的
// 一个 g 可能会在多个等待队列中，所以一个 g 可能被打包为多个 sudog
// 多个 g 也可以等待在同一个同步对象上
// 因此对于一个同步对象就会有很多 sudog 了
// sudog 是从一个特殊的池中进行分配的。用 acquireSudog 和 releaseSudog 来分配和释放 sudog
type sudog struct {

    // 之后的这些字段都是被该 g 所挂在的 channel 中的 hchan.lock 来保护的
    // shrinkstack depends on
    // this for sudogs involved in channel ops.
    g *g

    // isSelect 表示一个 g 是否正在参与 select 操作
    // 所以 g.selectDone 必须用 CAS 来操作，以胜出唤醒的竞争
    isSelect bool
    next     *sudog
    prev     *sudog
    elem     unsafe.Pointer // data element (may point to stack)

    // 下面这些字段则永远都不会被并发访问
    // 对于 channel 来说，waitlink 只会被 g 访问
    // 对于信号量来说，所有的字段，包括上面的那些字段都只在持有 semaRoot 锁时才可以访问
    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel
}
```

线程在 runtime 中的结构，对应一个 pthread，pthread 也会对应唯一的内核线程(task_struct):

```go
type m struct {
    g0      *g     // 用来执行调度指令的 goroutine
    morebuf gobuf  // gobuf arg to morestack
    divmod  uint32 // div/mod denominator for arm - known to liblink

    // Fields not known to debuggers.
    procid        uint64       // for debuggers, but offset not hard-coded
    gsignal       *g           // signal-handling g
    goSigStack    gsignalStack // Go-allocated signal handling stack
    sigmask       sigset       // storage for saved signal mask
    tls           [6]uintptr   // thread-local storage (for x86 extern register)
    mstartfn      func()
    curg          *g       // 当前运行的用户 goroutine
    caughtsig     guintptr // goroutine running during fatal signal
    p             puintptr // attached p for executing go code (nil if not executing go code)
    nextp         puintptr
    id            int64
    mallocing     int32
    throwing      int32
    preemptoff    string // 该字段不等于空字符串的话，要保持 curg 始终在这个 m 上运行
    locks         int32
    softfloat     int32
    dying         int32
    profilehz     int32
    helpgc        int32
    spinning      bool // m 失业了，正在积极寻找工作~
    blocked       bool // m 正阻塞在 note 上
    inwb          bool // m 正在执行 write barrier
    newSigstack   bool // minit on C thread called sigaltstack
    printlock     int8
    incgo         bool   // m 正在执行 cgo call
    freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
    fastrand      [2]uint32
    needextram    bool
    traceback     uint8
    ncgocall      uint64      // cgo 调用总计数
    ncgo          int32       // 当前正在执行的 cgo 订单计数
    cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
    cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
    park          note
    alllink       *m // on allm
    schedlink     muintptr
    mcache        *mcache
    lockedg       guintptr
    createstack   [32]uintptr    // stack that created this thread.
    freglo        [16]uint32     // d[i] lsb and f[i]
    freghi        [16]uint32     // d[i] msb and f[i+16]
    fflag         uint32         // floating point compare flags
    lockedExt     uint32         // tracking for external LockOSThread
    lockedInt     uint32         // tracking for internal lockOSThread
    nextwaitm     muintptr       // 正在等待锁的下一个 m
    waitunlockf   unsafe.Pointer // todo go func(*g, unsafe.pointer) bool
    waitlock      unsafe.Pointer
    waittraceev   byte
    waittraceskip int
    startingtrace bool
    syscalltick   uint32
    thread        uintptr // thread handle
    freelink      *m      // on sched.freem

    // these are here because they are too large to be on the stack
    // of low-level NOSPLIT functions.
    libcall   libcall
    libcallpc uintptr // for cpu profiler
    libcallsp uintptr
    libcallg  guintptr
    syscall   libcall // 存储 windows 平台的 syscall 参数

    mOS
}
```

抽象数据结构，可以认为是 processor 的抽象，代表了任务执行时的上下文，m 必须获得 p 才能执行:

```go
type p struct {
    lock mutex

    id          int32
    status      uint32 // one of pidle/prunning/...
    link        puintptr
    schedtick   uint32     // 每次调用 schedule 时会加一
    syscalltick uint32     // 每次系统调用时加一
    sysmontick  sysmontick // 上次 sysmon 观察到的 tick 时间
    m           muintptr   // 和相关联的 m 的反向指针，如果 p 是 idle 的话，那这个指针是 nil
    mcache      *mcache
    racectx     uintptr

    deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
    deferpoolbuf [5][32]*_defer

    // Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
    goidcache    uint64
    goidcacheend uint64

    // runnable 状态的 goroutine。访问时是不加锁的
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    // runnext 非空时，代表的是一个 runnable 状态的 G，
    // 这个 G 是被 当前 G 修改为 ready 状态的，
    // 并且相比在 runq 中的 G 有更高的优先级
    // 如果当前 G 的还有剩余的可用时间，那么就应该运行这个 G
    // 运行之后，该 G 会继承当前 G 的剩余时间
    // If a set of goroutines is locked in a
    // communicate-and-wait pattern, this schedules that set as a
    // unit and eliminates the (potentially large) scheduling
    // latency that otherwise arises from adding the ready'd
    // goroutines to the end of the run queue.
    runnext guintptr

    // Available G's (status == Gdead)
    gfree    *g
    gfreecnt int32

    sudogcache []*sudog
    sudogbuf   [128]*sudog

    tracebuf traceBufPtr

    // traceSweep indicates the sweep events should be traced.
    // This is used to defer the sweep start event until a span
    // has actually been swept.
    traceSweep bool
    // traceSwept and traceReclaimed track the number of bytes
    // swept and reclaimed by sweeping in the current sweep loop.
    traceSwept, traceReclaimed uintptr

    palloc persistentAlloc // per-P to avoid mutex

    // Per-P GC state
    gcAssistTime         int64 // Nanoseconds in assistAlloc
    gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker
    gcBgMarkWorker       guintptr
    gcMarkWorkerMode     gcMarkWorkerMode

    // 当前标记 worker 的开始时间，单位纳秒
    gcMarkWorkerStartTime int64

    // gcw is this P's GC work buffer cache. The work buffer is
    // filled by write barriers, drained by mutator assists, and
    // disposed on certain GC state transitions.
    gcw gcWork

    // wbBuf is this P's GC write barrier buffer.
    //
    // TODO: Consider caching this in the running G.
    wbBuf wbBuf

    runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

    pad [sys.CacheLineSize]byte
}
```

全局调度器，全局只有一个 schedt 类型的实例:

```go
type schedt struct {
    // 下面两个变量需以原子访问访问。保持在 struct 顶部，以使其在 32 位系统上可以对齐
    goidgen  uint64
    lastpoll uint64

    lock mutex

    // 当修改 nmidle，nmidlelocked，nmsys，nmfreed 这些数值时
    // 需要记得调用 checkdead

    midle        muintptr // idle m's waiting for work
    nmidle       int32    // 当前等待工作的空闲 m 计数
    nmidlelocked int32    // 当前等待工作的被 lock 的 m 计数
    mnext        int64    // 当前预缴创建的 m 数，并且该值会作为下一个创建的 m 的 ID
    maxmcount    int32    // 允许创建的最大的 m 数量
    nmsys        int32    // number of system m's not counted for deadlock
    nmfreed      int64    // cumulative number of freed m's

    ngsys uint32 // number of system goroutines; updated atomically

    pidle      puintptr // 空闲 p's
    npidle     uint32
    nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

    // 全局的可运行 g 队列
    runqhead guintptr
    runqtail guintptr
    runqsize int32

    // dead G 的全局缓存
    gflock       mutex
    gfreeStack   *g
    gfreeNoStack *g
    ngfree       int32

    // sudog 结构的集中缓存
    sudoglock  mutex
    sudogcache *sudog

    // 不同大小的可用的 defer struct 的集中缓存池
    deferlock mutex
    deferpool [5]*_defer

    // 被设置了 m.exited 标记之后的 m，这些 m 正在 freem 这个链表上等待被 free
    // 链表用 m.freelink 字段进行链接
    freem *m

    gcwaiting  uint32 // gc is waiting to run
    stopwait   int32
    stopnote   note
    sysmonwait uint32
    sysmonnote note

    // safepointFn should be called on each P at the next GC
    // safepoint if p.runSafePointFn is set.
    safePointFn   func(*p)
    safePointWait int32
    safePointNote note

    profilehz int32 // cpu profiling rate

    procresizetime int64 // 上次修改 gomaxprocs 的纳秒时间
    totaltime      int64 // ∫gomaxprocs dt up to procresizetime
}
```

## g/p/m 的关系

Go 实现了所谓的 M:N 模型，执行用户代码的 goroutine 可以认为都是对等的 goroutine。不考虑 g0 和 gsignal 的话，我们可以简单地认为调度就是将 m 绑定到 p，然后在 m 中不断循环执行调度函数(runtime.schedule)，寻找可用的 g 来执行，下图为 m 绑定到 p 时，可能得到的 g 的来源:

```
                                                +--------------+
                                                |    binded    +-------------------------------------+
                                                +-------+------+                                     |
+------------------------------------+                  |                                            v                         +------------------------------------+
|                                    |                  |                         +------------------------------------+       |                                    |
|             +------------------+   |                  |                         |                                    |       |            +------------------+    |
|             | Local Run Queue  |   |                  |                         |             +------------------+   |       |            | Global Run Queue |    |
|   other P   +-+-+-+-+-+-+-+-+--+   |                  |                         |             | Local Run Queue  |   |       |  schedt    +--+-+-+-+-+-+-+---+    |
|               |G|G|G|G|G|G|G|      |                  |                         |    P        +-+-+-+-+-+-+-+-+--+   |       |               |G|G|G|G|G|G|        |
|               +-+-+-+-+-+-+-+      |                  |                         |               |G|G|G|G|G|G|G|      |       |               +-+-+-+-+-+-+        |
|                ^                   |                  |                         |               +-+-+-+-+-+-+-+      |       |                ^                   |
+----------------+-------------------+                  |                         |                ^                   |       +----------------+-------------------+
                 |                                      |                         +----------------+-------------------+                        |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      |                                          |                                            |
                 |                                      v                                          |                                            |
          +------+-------+                             .-.      +----------------+                 |                                            |
          |    steal     +----------------------------( M )-----+    runqget     +-----------------+                                            |
          +--------------+                             `-'      +----------------+                                                              |
                                                        |                                                                                       |
                                                        |                                                                           +-----------+-----+
                                                        +---------------------------------------------------------------------------+   globrunqget   |
                                                        |                                                                           +-----------------+
                                                        |
                                                        |
                                                        |
                                                        |
                                                        |
                                                        |
                                             +----------+--------+
                                             |   get netpoll g   |
                                             +----------+--------+
                                                        |
                                                        |
                                                        |
                                                        |
                                                        |
                                         +--------------+--------------------+
                                         |              |                    |
                                         |              |                    |
                                         |   netpoll    v                    |
                                         |             +-+-+-+-+             |
                                         |             |G|G|G|G|             |
                                         |             +-+-+-+-+             |
                                         |                                   |
                                         +-----------------------------------+
```


这张图展示了 g、p、m 三者之间的大致关系。m 是执行实体，对应的是操作系统线程。可以看到 m 会从绑定的 p 的本地队列、sched 中的全局队列、netpoll 中获取可运行的 g，实在找不着还会去其它的 p 那里去偷。

## p 如何初始化

程序启动时，会依次调用：

```mermaid
graph TD
runtime.schedinit -->  runtime.procresize
```

在 procresize 中会将全局 p 数组初始化，并将这些 p 串成链表放进 sched 全局调度器的 pidle 队列中:

```go
for i := nprocs - 1; i >= 0; i-- {
    p := allp[i]

    // ...
    // 设置 p 的状态
    p.status = _Pidle
    // 初始化时，所有 p 的 runq 都是空的，所以一定会走这个 if
    if runqempty(p) {
        // 将 p 放到全局调度器的 pidle 队列中
        pidleput(p)
    } else {
        // ...
    }
}
```

pidleput 也比较简单，没啥可说的:

```go
func pidleput(_p_ *p) {
    if !runqempty(_p_) {
        throw("pidleput: P has non-empty run queue")
    }
    // 简单的链表操作
    _p_.link = sched.pidle
    sched.pidle.set(_p_)

    // pidle count + 1
    atomic.Xadd(&sched.npidle, 1)
}
```

所有 p 在程序启动的时候就已经被初始化完毕了，除非手动调用 runtime.GOMAXPROCS。

```go
func GOMAXPROCS(n int) int {
    lock(&sched.lock)
    ret := int(gomaxprocs)
    unlock(&sched.lock)
    if n <= 0 || n == ret {
        return ret
    }

    stopTheWorld("GOMAXPROCS")

    // newprocs will be processed by startTheWorld
    newprocs = int32(n)

    startTheWorld()
    return ret
}
```

在 startTheWorld 中会调用 procresize。

## g 如何创建

在用户代码里一般这么写:

```go
go func() {
    // do the stuff
}()
```

实际上会被翻译成 `runtime.newproc`，特权语法只是个语法糖。如果你要在其它语言里实现类似的东西，只要实现编译器翻译之后的内容就好了。具体流程:

```mermaid
graph TD
runtime.newproc --> runtime.newproc1
```

newproc 干的事情也比较简单

```go
func newproc(siz int32, fn *funcval) {
    // add 是一个指针运算，跳过函数指针
    // 把栈上的参数起始地址找到
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    pc := getcallerpc()
    systemstack(func() {
        newproc1(fn, (*uint8)(argp), siz, pc)
    })
}

// funcval 是一个变长结构，第一个成员是函数指针
// 所以上面的 add 是跳过这个 fn
type funcval struct {
    fn uintptr
    // variable-size, fn-specific data here
}
```

runtime 里比较常见的 getcallerpc 和 getcallersp，代码里的注释写的比较明白了:

```go
// For example:
//
// func f(arg1, arg2, arg3 int) {
//    pc := getcallerpc()
//    sp := getcallersp(unsafe.Pointer(&arg1))
//}
//
// These two lines find the PC and SP immediately following
// the call to f (where f will return).
//
```

getcallerpc 返回的是调用函数之后的那条程序指令的地址，即 callee 函数返回时要执行的下一条指令的地址。

systemstack 在 runtime 中用的也比较多，其功能为让 m 切换到 g0 上执行各种调度函数。至于啥是 g0，在讲 m 的时候再说。

newproc1 的工作流程也比较简单:

```mermaid
graph TD
newproc1 --> newg
newg[gfget] --> nil{is nil?}
nil -->|yes|E[init stack]
nil -->|no|C[malg]
C --> D[set g status=> idle->dead]
D --> allgadd
E --> G[set g status=> dead-> runnable]
allgadd --> G
G --> runqput
```

删掉了不关心的细节后的代码:

```go
func newproc1(fn *funcval, argp *uint8, narg int32, callerpc uintptr) {
    _g_ := getg()

    if fn == nil {
        _g_.m.throwing = -1 // do not dump full stacks
        throw("go of nil func value")
    }
    _g_.m.locks++ // disable preemption because it can be holding p in a local var
    siz := narg
    siz = (siz + 7) &^ 7


    _p_ := _g_.m.p.ptr()
    newg := gfget(_p_)
    if newg == nil {
        newg = malg(_StackMin)
        casgstatus(newg, _Gidle, _Gdead)
        allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
    }

    totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
    totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
    sp := newg.stack.hi - totalSize
    spArg := sp

    // 初始化 g，g 的 gobuf 现场，g 的 m 的 curg
    // 以及各种寄存器
    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    newg.sched.sp = sp
    newg.stktopsp = sp
    newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
    newg.sched.g = guintptr(unsafe.Pointer(newg))
    gostartcallfn(&newg.sched, fn)
    newg.gopc = callerpc
    newg.startpc = fn.fn
    if _g_.m.curg != nil {
        newg.labels = _g_.m.curg.labels
    }

    casgstatus(newg, _Gdead, _Grunnable)

    newg.goid = int64(_p_.goidcache)
    _p_.goidcache++
    runqput(_p_, newg, true)

    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
        wakep()
    }
    _g_.m.locks--
    if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
        _g_.stackguard0 = stackPreempt
    }
}
```

所以 `go func` 执行的结果是调用 runqput 将 g 放进了执行队列。什么时候开始执行并不是用户代码能决定得了的。再看看 runqput 这个函数:

```go
// runqput 常识把 g 放到本地执行队列中
// next 参数如果是 false 的话，runqput 会将 g 放到运行队列的尾部
// If next if false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(_p_ *p, gp *g, next bool) {
    if randomizeScheduler && next && fastrand()%2 == 0 {
        next = false
    }

    if next {
    retryNext:
        oldnext := _p_.runnext
        if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            goto retryNext
        }
        if oldnext == 0 {
            return
        }
        // 把之前的 runnext 踢到正常的 runq 中
        gp = oldnext.ptr()
    }

retry:
    h := atomic.Load(&_p_.runqhead) // load-acquire, synchronize with consumers
    t := _p_.runqtail
    if t-h < uint32(len(_p_.runq)) {
        _p_.runq[t%uint32(len(_p_.runq))].set(gp)
        atomic.Store(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
        return
    }
    if runqputslow(_p_, gp, h, t) {
        return
    }
    // 队列没有满的话，上面的 put 操作会成功
    goto retry
}
```

## m 工作机制

在 runtime 中有三种线程，一种是主线程，一种是用来跑 sysmon 的线程，一种是普通的用户线程。主线程在 runtime 由对应的全局变量: `runtime.m0` 来表示。用户线程就是普通的线程了，和 p 绑定，执行 g 中的任务。虽然说是有三种，实际上前两种线程整个 runtime 就只有一个实例。用户线程才会有很多实例。

### 主线程 m0

主线程中用来跑 `runtime.main`，流程线性执行，没有跳转:

```mermaid
graph TD
runtime.main --> A[init max stack size]
A --> B[systemstack execute -> newm -> sysmon]
B --> runtime.lockOsThread
runtime.lockOsThread --> runtime.init
runtime.init --> runtime.gcenable
runtime.gcenable --> main.init
main.init --> main.main
```

### sysmon 线程

sysmon 是在 `runtime.main` 中启动的，不过需要注意的是 sysmon 并不是在 m0 上执行的。因为:

```go
systemstack(func() {
    newm(sysmon, nil)
})
```

创建了新的 m，但这个 m 又与普通的线程不一样，因为不需要绑定 p 就可以执行。是与整个调度系统脱离的。

sysmon 内部是个死循环，主要负责以下几件事情:

1. checkdead，检查是否所有 goroutine 都已经锁死，如果是的话，直接调用 runtime.throw，强制退出。这个操作只在启动的时候做一次

2. 将 netpoll 返回的结果注入到全局 sched 的任务队列

3. 收回因为 syscall 而长时间阻塞的 p，同时抢占那些执行时间过长的 g

4. 如果 span 内存闲置超过 5min，那么释放掉

流程图:

```mermaid
graph TD
sysmon --> usleep
usleep --> checkdead
checkdead --> |every 10ms|C[netpollinited && lastpoll != 0]
C --> |yes|netpoll
netpoll --> injectglist
injectglist --> retake
C --> |no|retake
retake --> A[check forcegc needed]
A --> B[scavenge heap once in a while]
B --> usleep

```

```go
// sysmon 不需要绑定 P 就可以运行，所以不允许 write barriers
//
//go:nowritebarrierrec
func sysmon() {
    lock(&sched.lock)
    sched.nmsys++
    checkdead()
    unlock(&sched.lock)

    // 如果一个 heap span 在一次GC 之后 5min 都没有被使用过
    // 那么把它交还给操作系统
    scavengelimit := int64(5 * 60 * 1e9)

    if debug.scavenge > 0 {
        // Scavenge-a-lot for testing.
        forcegcperiod = 10 * 1e6
        scavengelimit = 20 * 1e6
    }

    lastscavenge := nanotime()
    nscavenge := 0

    lasttrace := int64(0)
    idle := 0 // how many cycles in succession we had not wokeup somebody
    delay := uint32(0)
    for {
        if idle == 0 { // 初始化时 20us sleep
            delay = 20
        } else if idle > 50 { // start doubling the sleep after 1ms...
            delay *= 2
        }
        if delay > 10*1000 { // 最多到 10ms
            delay = 10 * 1000
        }
        usleep(delay)
        if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
            lock(&sched.lock)
            if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
                atomic.Store(&sched.sysmonwait, 1)
                unlock(&sched.lock)
                // Make wake-up period small enough
                // for the sampling to be correct.
                maxsleep := forcegcperiod / 2
                if scavengelimit < forcegcperiod {
                    maxsleep = scavengelimit / 2
                }
                shouldRelax := true
                if osRelaxMinNS > 0 {
                    next := timeSleepUntil()
                    now := nanotime()
                    if next-now < osRelaxMinNS {
                        shouldRelax = false
                    }
                }
                if shouldRelax {
                    osRelax(true)
                }
                notetsleep(&sched.sysmonnote, maxsleep)
                if shouldRelax {
                    osRelax(false)
                }
                lock(&sched.lock)
                atomic.Store(&sched.sysmonwait, 0)
                noteclear(&sched.sysmonnote)
                idle = 0
                delay = 20
            }
            unlock(&sched.lock)
        }
        // trigger libc interceptors if needed
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        // 如果 10ms 没有 poll 过 network，那么就 netpoll 一次
        lastpoll := int64(atomic.Load64(&sched.lastpoll))
        now := nanotime()
        if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
            atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
            gp := netpoll(false) // 非阻塞 -- 返回一个 goroutine 的列表
            if gp != nil {
                // Need to decrement number of idle locked M's
                // (pretending that one more is running) before injectglist.
                // Otherwise it can lead to the following situation:
                // injectglist grabs all P's but before it starts M's to run the P's,
                // another M returns from syscall, finishes running its G,
                // observes that there is no work to do and no other running M's
                // and reports deadlock.
                incidlelocked(-1)
                injectglist(gp)
                incidlelocked(1)
            }
        }
        // 接收在 syscall 状态阻塞的 P
        // 抢占长时间运行的 G
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }
        // 检查是否需要 force GC(两分钟一次的)
        if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
            lock(&forcegc.lock)
            forcegc.idle = 0
            forcegc.g.schedlink = 0
            injectglist(forcegc.g)
            unlock(&forcegc.lock)
        }
        // 每过一段时间扫描一次堆
        if lastscavenge+scavengelimit/2 < now {
            mheap_.scavenge(int32(nscavenge), uint64(now), uint64(scavengelimit))
            lastscavenge = now
            nscavenge++
        }
        if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
            lasttrace = now
            schedtrace(debug.scheddetail > 0)
        }
    }
}
```

checkdead:

```go
// 检查死锁的场景
// 该检查基于当前正在运行的 M 的数量，如果 0，那么就是 deadlock 了
// 检查的时候必须持有 sched.lock 锁
func checkdead() {
    // 对于 -buildmode=c-shared 或者 -buildmode=c-archive 来说
    // 没有 goroutine 正在运行也是 OK 的。因为调用这个库的程序应该是在运行的
    if islibrary || isarchive {
        return
    }

    // If we are dying because of a signal caught on an already idle thread,
    // freezetheworld will cause all running threads to block.
    // And runtime will essentially enter into deadlock state,
    // except that there is a thread that will call exit soon.
    if panicking > 0 {
        return
    }

    run := mcount() - sched.nmidle - sched.nmidlelocked - sched.nmsys
    if run > 0 {
        return
    }
    if run < 0 {
        print("runtime: checkdead: nmidle=", sched.nmidle, " nmidlelocked=", sched.nmidlelocked, " mcount=", mcount(), " nmsys=", sched.nmsys, "\n")
        throw("checkdead: inconsistent counts")
    }

    grunning := 0
    lock(&allglock)
    for i := 0; i < len(allgs); i++ {
        gp := allgs[i]
        if isSystemGoroutine(gp) {
            continue
        }
        s := readgstatus(gp)
        switch s &^ _Gscan {
        case _Gwaiting:
            grunning++
        case _Grunnable,
            _Grunning,
            _Gsyscall:
            unlock(&allglock)
            print("runtime: checkdead: find g ", gp.goid, " in status ", s, "\n")
            throw("checkdead: runnable g")
        }
    }
    unlock(&allglock)
    if grunning == 0 { // possible if main goroutine calls runtime·Goexit()
        throw("no goroutines (main called runtime.Goexit) - deadlock!")
    }

    // Maybe jump time forward for playground.
    gp := timejump()
    if gp != nil {
        casgstatus(gp, _Gwaiting, _Grunnable)
        globrunqput(gp)
        _p_ := pidleget()
        if _p_ == nil {
            throw("checkdead: no p for timer")
        }
        mp := mget()
        if mp == nil {
            // There should always be a free M since
            // nothing is running.
            throw("checkdead: no m for timer")
        }
        mp.nextp.set(_p_)
        notewakeup(&mp.park)
        return
    }

    getg().m.throwing = -1 // do not dump full stacks
    throw("all goroutines are asleep - deadlock!")
}
```

retake:

```go
// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
    n := 0
    // Prevent allp slice changes. This lock will be completely
    // uncontended unless we're already stopping the world.
    lock(&allpLock)
    // We can't use a range loop over allp because we may
    // temporarily drop the allpLock. Hence, we need to re-fetch
    // allp each time around the loop.
    for i := 0; i < len(allp); i++ {
        _p_ := allp[i]
        if _p_ == nil {
            // 在 procresize 修改了 allp 但还没有创建新的 p 的时候
            // 会有这种情况
            continue
        }
        pd := &_p_.sysmontick
        s := _p_.status
        if s == _Psyscall {
            // 从 syscall 接管 P，如果它进行 syscall 已经经过了一个 sysmon 的 tick(至少 20us)
            t := int64(_p_.syscalltick)
            if int64(pd.syscalltick) != t {
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
            // 一方面如果没有其它工作可做的话，我们不想接管 p
            // 但另一方面为了避免 sysmon 线程陷入沉睡，我们最终还是会接管这些 p
            if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            // 解开 allplock 的锁，然后就可以持有 sched.lock 锁了
            unlock(&allpLock)
            // Need to decrement number of idle locked M's
            // (pretending that one more is running) before the CAS.
            // Otherwise the M from which we retake can exit the syscall,
            // increment nmidle and report deadlock.
            incidlelocked(-1)
            if atomic.Cas(&_p_.status, s, _Pidle) {
                if trace.enabled {
                    traceGoSysBlock(_p_)
                    traceProcStop(_p_)
                }
                n++
                _p_.syscalltick++
                handoffp(_p_)
            }
            incidlelocked(1)
            lock(&allpLock)
        } else if s == _Prunning {
            // 如果 G 运行时间太长，那么抢占它
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
            if pd.schedwhen+forcePreemptNS > now {
                continue
            }
            preemptone(_p_)
        }
    }
    unlock(&allpLock)
    return uint32(n)
}
```

### 普通线程

普通线程就是我们 G/P/M 模型里的 M 了，M 对应的就是操作系统的线程。

#### 线程创建

上面在创建 sysmon 线程的时候也看到了，创建线程的函数是 newm。

```mermaid
graph TD
newm --> newm1
newm1 --> newosproc
newosproc --> clone
```

最终会走到 linux 创建线程的系统调用 `clone`，代码里大段和 cgo 相关的内容我们就不关心了，摘掉 cgo 相关的逻辑后的代码如下:

```go
// 创建一个新的 m。该 m 会在启动时调用函数 fn，或者 schedule 函数
// fn 需要是 static 类型，且不能是在堆上分配的闭包。
// 运行 m 时，m.p 是有可能为 nil 的，所以不允许 write barriers
//go:nowritebarrierrec
func newm(fn func(), _p_ *p) {
    mp := allocm(_p_, fn)
    mp.nextp.set(_p_)
    mp.sigmask = initSigmask
    newm1(mp)
}
```

传入的 p 会被赋值给 m 的 nextp 成员，在 m 执行 schedule 时，会将 nextp 拿出来，进行之后真正的绑定操作(其实就是把 nextp 赋值为 nil，并把这个 nextp 赋值给 m.p，把 m 赋值给 p.m)。

```go
func newm1(mp *m) {
    execLock.rlock() // Prevent process clone.
    newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))
    execLock.runlock()
}
```

```go
func newosproc(mp *m, stk unsafe.Pointer) {
    // Disable signals during clone, so that the new thread starts
    // with signals disabled. It will enable them in minit.
    var oset sigset
    sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
    ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
    sigprocmask(_SIG_SETMASK, &oset, nil)

    if ret < 0 {
        print("runtime: failed to create new OS thread (have ", mcount(), " already; errno=", -ret, ")\n")
        if ret == -_EAGAIN {
            println("runtime: may need to increase max user processes (ulimit -u)")
        }
        throw("newosproc")
    }
}
```

#### 工作流程

首先空闲的 m 会被丢进全局调度器的 midle 队列中，在需要 m 的时候，会先从这里取:

```go
//go:nowritebarrierrec
// 常识从 midle 列表中获取一个 m
// 必须锁全局的 sched
// 可能在 STW 期间执行，所以不允许 write barriers
func mget() *m {
    mp := sched.midle.ptr()
    if mp != nil {
        sched.midle = mp.schedlink
        sched.nmidle--
    }
    return mp
}
```

取不到的话就会调用之前提到的 newm 来创建新线程，创建的线程是不会被销毁的，哪怕之后不需要这么多 m 了，也就只是会把 m 放在 midle 中。

什么时候会创建线程呢，可以追踪一下 newm 的调用方:

```mermaid
graph TD
main --> |sysmon|newm
startTheWorld --> startTheWorldWithSema
gcMarkTermination --> startTheWorldWithSema
gcStart--> startTheWorldWithSema
startTheWorldWithSema --> |helpgc|newm
startTheWorldWithSema --> |run p|newm
startm --> mget
mget --> |if no free m|newm
startTemplateThread --> |templateThread|newm
LockOsThread --> startTemplateThread
main --> |iscgo|startTemplateThread
handoffp --> startm
wakep --> startm
injectglist --> startm
```

基本上来讲，m 都是按需创建的。如果 sched.midle 中没有空闲的 m 了，现在又需要，那么就会去创建一个。

创建好的线程需要绑定到 p 之后才会开始执行，执行过程中也可能被剥夺掉 p。比如:

TODO

工作线程执行的内容核心其实就只有俩: `schedule()` 和 `findrunnable()`。

#### schedule

```mermaid
graph TD
schedule --> A[schedtick%61 == 0]
A --> |yes|globrunqget
A --> |no|runqget
globrunqget --> C[gp == nil]
C --> |no|execute
C --> |yes|runqget
runqget --> B[gp == nil]
B --> |no|execute
B --> |yes|findrunnable
findrunnable --> execute
```

```go
// 调度器调度一轮要执行的函数: 寻找一个 runnable 状态的 goroutine，并 execute 它
// 调度函数是循环，永远都不会返回
func schedule() {
    _g_ := getg()

    if _g_.m.locks != 0 {
        throw("schedule: holding locks")
    }

    if _g_.m.lockedg != 0 {
        stoplockedm()
        execute(_g_.m.lockedg.ptr(), false) // Never returns.
    }

    // 执行 cgo 调用的 g 不能被 schedule 走
    // 因为 cgo 调用使用 m 的 g0 栈
    if _g_.m.incgo {
        throw("schedule: in cgo")
    }

top:
    if sched.gcwaiting != 0 {
        gcstopm()
        goto top
    }
    if _g_.m.p.ptr().runSafePointFn != 0 {
        runSafePointFn()
    }

    var gp *g
    var inheritTime bool
    if trace.enabled || trace.shutdown {
        gp = traceReader()
        if gp != nil {
            casgstatus(gp, _Gwaiting, _Grunnable)
            traceGoUnpark(gp, 0)
        }
    }
    if gp == nil && gcBlackenEnabled != 0 {
        gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
    }
    if gp == nil {
        // 每调度几次就检查一下全局的 runq 来确保公平
        // 否则两个 goroutine 就可以通过互相调用
        // 完全占用本地的 runq 了
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            gp = globrunqget(_g_.m.p.ptr(), 1)
            unlock(&sched.lock)
        }
    }
    if gp == nil {
        gp, inheritTime = runqget(_g_.m.p.ptr())
        if gp != nil && _g_.m.spinning {
            throw("schedule: spinning with local work")
        }
    }
    if gp == nil {
        gp, inheritTime = findrunnable() // 在找到 goroutine 之前会一直阻塞下去
    }

    // 当前线程将要执行 goroutine，并且不会再进入 spinning 状态
    // 所以如果它被标记为 spinning，我们需要 reset 这个状态
    // 可能会重启一个新的 spinning 状态的 M
    if _g_.m.spinning {
        resetspinning()
    }

    if gp.lockedm != 0 {
        // Hands off own p to the locked m,
        // then blocks waiting for a new p.
        startlockedm(gp)
        goto top
    }

    execute(gp, inheritTime)
}
```

m 中所谓的调度循环实际上就是一直在执行下图中的 loop:

```mermaid
graph TD
schedule --> execute
execute --> gogo
gogo --> goexit1
goexit1 --> goexit0
goexit0 --> schedule
```

#### findrunnable

findrunnable 比较复杂，流程图先把 gc 相关的省略掉了:

```mermaid
graph TD
runqget --> A[gp == nil]
A --> |no|return
A --> |yes|globrunqget
globrunqget --> B[gp == nil]
B --> |no| return
B --> |yes| C[netpollinited && lastpoll != 0]
C --> |yes|netpoll
netpoll --> K[gp == nil]
K --> |no|return
K --> |yes|runqsteal
C --> |no|runqsteal
runqsteal --> D[gp == nil]
D --> |no|return
D --> |yes|E[globrunqget]
E --> F[gp == nil]
F --> |no| return
F --> |yes| G[check all p's runq]
G --> H[runq is empty]
H --> |no|runqget
H --> |yes|I[netpoll]
I --> J[gp == nil]
J --> |no| return
J --> |yes| stopm
stopm --> runqget
```

```go
// 找到一个可执行的 goroutine 来 execute
// 会尝试从其它的 P 那里偷 g，从全局队列中拿，或者 network 中 poll
func findrunnable() (gp *g, inheritTime bool) {
    _g_ := getg()

    // The conditions here and in handoffp must agree: if
    // findrunnable would return a G to run, handoffp must start
    // an M.

top:
    _p_ := _g_.m.p.ptr()
    if sched.gcwaiting != 0 {
        gcstopm()
        goto top
    }
    if _p_.runSafePointFn != 0 {
        runSafePointFn()
    }
    if fingwait && fingwake {
        if gp := wakefing(); gp != nil {
            ready(gp, 0, true)
        }
    }
    if *cgo_yield != nil {
        asmcgocall(*cgo_yield, nil)
    }

    // 本地 runq
    if gp, inheritTime := runqget(_p_); gp != nil {
        return gp, inheritTime
    }

    // 全局 runq
    if sched.runqsize != 0 {
        lock(&sched.lock)
        gp := globrunqget(_p_, 0)
        unlock(&sched.lock)
        if gp != nil {
            return gp, false
        }
    }

    // Poll network.
    // netpoll 是我们执行 work-stealing 之前的一个优化
    // 如果没有任何的 netpoll 等待者，或者线程被阻塞在 netpoll 中，我们可以安全地跳过这段逻辑
    // 如果在阻塞的线程中存在任何逻辑上的竞争(e.g. 已经从 netpoll 中返回，但还没有设置 lastpoll)
    // 该线程还是会将下面的 netpoll 阻塞住
    if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        if gp := netpoll(false); gp != nil { // 非阻塞
            // netpoll 返回 goroutine 链表，用 schedlink 连接
            injectglist(gp.schedlink.ptr())
            casgstatus(gp, _Gwaiting, _Grunnable)
            if trace.enabled {
                traceGoUnpark(gp, 0)
            }
            return gp, false
        }
    }

    // 从其它 p 那里偷 g
    procs := uint32(gomaxprocs)
    if atomic.Load(&sched.npidle) == procs-1 {
        // GOMAXPROCS=1 或者除了我们其它的 p 都是 idle
        // 新的工作可能从 syscall/cgocall，网络或者定时器中来。
        // 上面这些任务都不会被放到本地的 runq，所有没有可以 stealing 的点
        goto stop
    }
    // 如果正在自旋的 M 的数量 >= 忙着的 P，那么阻塞
    // 这是为了
    // 当 GOMAXPROCS 远大于 1，但程序的并行度又很低的时候
    // 防止过量的 CPU 消耗
    if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
        goto stop
    }
    if !_g_.m.spinning {
        _g_.m.spinning = true
        atomic.Xadd(&sched.nmspinning, 1)
    }
    for i := 0; i < 4; i++ {
        for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            if sched.gcwaiting != 0 {
                goto top
            }
            stealRunNextG := i > 2 // first look for ready queues with more than 1 g
            if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
                return gp, false
            }
        }
    }

stop:

    // 没有可以干的事情。如果我们正在 GC 的标记阶段，可以安全地扫描和加深对象的颜色，
    // 这样可以进行空闲时间的标记，而不是直接放弃 P
    if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
        _p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
        gp := _p_.gcBgMarkWorker.ptr()
        casgstatus(gp, _Gwaiting, _Grunnable)
        if trace.enabled {
            traceGoUnpark(gp, 0)
        }
        return gp, false
    }

    // Before we drop our P, make a snapshot of the allp slice,
    // which can change underfoot once we no longer block
    // safe-points. We don't need to snapshot the contents because
    // everything up to cap(allp) is immutable.
    allpSnapshot := allp

    // 返回 P 并阻塞
    lock(&sched.lock)
    if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
        unlock(&sched.lock)
        goto top
    }
    if sched.runqsize != 0 {
        gp := globrunqget(_p_, 0)
        unlock(&sched.lock)
        return gp, false
    }
    if releasep() != _p_ {
        throw("findrunnable: wrong p")
    }
    pidleput(_p_)
    unlock(&sched.lock)

    // Delicate dance: thread transitions from spinning to non-spinning state,
    // potentially concurrently with submission of new goroutines. We must
    // drop nmspinning first and then check all per-P queues again (with
    // #StoreLoad memory barrier in between). If we do it the other way around,
    // another thread can submit a goroutine after we've checked all run queues
    // but before we drop nmspinning; as the result nobody will unpark a thread
    // to run the goroutine.
    // If we discover new work below, we need to restore m.spinning as a signal
    // for resetspinning to unpark a new worker thread (because there can be more
    // than one starving goroutine). However, if after discovering new work
    // we also observe no idle Ps, it is OK to just park the current thread:
    // the system is fully loaded so no spinning threads are required.
    // Also see "Worker thread parking/unparking" comment at the top of the file.
    wasSpinning := _g_.m.spinning
    if _g_.m.spinning {
        _g_.m.spinning = false
        if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
            throw("findrunnable: negative nmspinning")
        }
    }

    // 再检查一下所有的 runq
    for _, _p_ := range allpSnapshot {
        if !runqempty(_p_) {
            lock(&sched.lock)
            _p_ = pidleget()
            unlock(&sched.lock)
            if _p_ != nil {
                acquirep(_p_)
                if wasSpinning {
                    _g_.m.spinning = true
                    atomic.Xadd(&sched.nmspinning, 1)
                }
                goto top
            }
            break
        }
    }

    // 再检查 gc 空闲 g
    if gcBlackenEnabled != 0 && gcMarkWorkAvailable(nil) {
        lock(&sched.lock)
        _p_ = pidleget()
        if _p_ != nil && _p_.gcBgMarkWorker == 0 {
            pidleput(_p_)
            _p_ = nil
        }
        unlock(&sched.lock)
        if _p_ != nil {
            acquirep(_p_)
            if wasSpinning {
                _g_.m.spinning = true
                atomic.Xadd(&sched.nmspinning, 1)
            }
            // Go back to idle GC check.
            goto stop
        }
    }

    // poll network
    if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
        if _g_.m.p != 0 {
            throw("findrunnable: netpoll with p")
        }
        if _g_.m.spinning {
            throw("findrunnable: netpoll with spinning")
        }
        gp := netpoll(true) // 阻塞到返回为止
        atomic.Store64(&sched.lastpoll, uint64(nanotime()))
        if gp != nil {
            lock(&sched.lock)
            _p_ = pidleget()
            unlock(&sched.lock)
            if _p_ != nil {
                acquirep(_p_)
                injectglist(gp.schedlink.ptr())
                casgstatus(gp, _Gwaiting, _Grunnable)
                if trace.enabled {
                    traceGoUnpark(gp, 0)
                }
                return gp, false
            }
            injectglist(gp)
        }
    }
    stopm()
    goto top
}
```

## m 和 p 解绑定

handoffp:

```mermaid
graph TD

mexit --> A[is m0?]
A --> |yes|B[handoffp]
A --> |no| C[iterate allm]
C --> |m found|handoffp
C --> |m not found| throw

forEachP --> |p status == syscall| handoffp

stoplockedm --> handoffp

entersyscallblock --> entersyscallblock_handoff
entersyscallblock_handoff --> handoffp

retake --> |p status == syscall| handoffp
```

## g 的状态迁移

```mermaid
graph TD
start{newg} --> Gidle
Gidle --> |oneNewExtraM|Gdead
Gidle --> |newproc1|Gdead

Gdead --> |newproc1|Grunnable
Gdead --> |needm|Gsyscall

Gscanrunning --> |scang|Grunning

Grunnable --> |execute|Grunning

Gany --> |casgcopystack|Gcopystack

Gcopystack --> |todotodo|Grunning

Gsyscall --> |dropm|Gdead
Gsyscall --> |exitsyscall0|Grunnable
Gsyscall --> |exitsyscall|Grunning

Grunning --> |goschedImpl|Grunnable
Grunning --> |goexit0|Gdead
Grunning --> |newstack|Gcopystack
Grunning --> |reentersyscall|Gsyscall
Grunning --> |entersyscallblock|Gsyscall
Grunning --> |markroot|Gwaiting
Grunning --> |gcAssistAlloc1|Gwaiting
Grunning --> |park_m|Gwaiting
Grunning --> |gcMarkTermination|Gwaiting
Grunning --> |gcBgMarkWorker|Gwaiting
Grunning --> |newstack|Gwaiting

Gwaiting --> |gcMarkTermination|Grunning
Gwaiting --> |gcBgMarkWorker|Grunning
Gwaiting --> |markroot|Grunning
Gwaiting --> |gcAssistAlloc1|Grunning
Gwaiting --> |newstack|Grunning
Gwaiting --> |findRunnableGCWorker|Grunnable
Gwaiting --> |ready|Grunnable
Gwaiting --> |findrunnable|Grunnable
Gwaiting --> |injectglist|Grunnable
Gwaiting --> |schedule|Grunnable
Gwaiting --> |park_m|Grunnable
Gwaiting --> |procresize|Grunnable
Gwaiting --> |checkdead|Grunnable

```

## p 的状态迁移
