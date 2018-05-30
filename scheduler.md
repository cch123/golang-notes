# 调度

## 基本数据结构

goroutine 在 runtime 中的数据结构:

```go
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
type stack struct {
    lo uintptr
    hi uintptr
}

type g struct {
    // Stack parameters.
    // stack describes the actual stack memory: [stack.lo, stack.hi).
    // stackguard0 is the stack pointer compared in the Go stack growth prologue.
    // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
    // stackguard1 is the stack pointer compared in the C stack growth prologue.
    // It is stack.lo+StackGuard on g0 and gsignal stacks.
    // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
    stack       stack   // offset known to runtime/cgo
    stackguard0 uintptr // offset known to liblink
    stackguard1 uintptr // offset known to liblink

    _panic         *_panic // innermost panic - offset known to liblink
    _defer         *_defer // innermost defer
    m              *m      // current m; offset known to arm liblink
    sched          gobuf
    syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
    syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
    stktopsp       uintptr        // expected sp at top of stack, to check in traceback
    param          unsafe.Pointer // passed parameter on wakeup
    atomicstatus   uint32
    stackLock      uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
    goid           int64
    waitsince      int64  // approx time when the g become blocked
    waitreason     string // if status==Gwaiting
    schedlink      guintptr
    preempt        bool     // preemption signal, duplicates stackguard0 = stackpreempt
    throwsplit     bool     // must not split stack
    raceignore     int8     // ignore race detection events
    sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
    sysexitticks   int64    // cputicks when syscall has returned (for tracing)
    traceseq       uint64   // trace event sequencer
    tracelastp     puintptr // last P emitted an event for this goroutine
    lockedm        muintptr
    sig            uint32
    writebuf       []byte
    sigcode0       uintptr
    sigcode1       uintptr
    sigpc          uintptr
    gopc           uintptr // pc of go statement that created this goroutine
    startpc        uintptr // pc of goroutine function
    racectx        uintptr
    waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
    cgoCtxt        []uintptr      // cgo traceback context
    labels         unsafe.Pointer // profiler labels
    timer          *timer         // cached timer for time.Sleep
    selectDone     uint32         // are we participating in a select and did someone win the race?
}
```

当 g 遇到阻塞，或需要等待的场景时，会被打包成 sudog 这样一个结构。一个 g 可能被打包为多个 sudog 分别挂在不同的等待队列上:

```go
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
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

线程在 runtime 中的结构，对应一个 pthread，pthread 也会对应唯一的内核线程(task_struct):

```go
type m struct {
    g0      *g     // goroutine with scheduling stack
    morebuf gobuf  // gobuf arg to morestack
    divmod  uint32 // div/mod denominator for arm - known to liblink

    // Fields not known to debuggers.
    procid        uint64       // for debuggers, but offset not hard-coded
    gsignal       *g           // signal-handling g
    goSigStack    gsignalStack // Go-allocated signal handling stack
    sigmask       sigset       // storage for saved signal mask
    tls           [6]uintptr   // thread-local storage (for x86 extern register)
    mstartfn      func()
    curg          *g       // current running goroutine
    caughtsig     guintptr // goroutine running during fatal signal
    p             puintptr // attached p for executing go code (nil if not executing go code)
    nextp         puintptr
    id            int64
    mallocing     int32
    throwing      int32
    preemptoff    string // if != "", keep curg running on this m
    locks         int32
    softfloat     int32
    dying         int32
    profilehz     int32
    helpgc        int32
    spinning      bool // m is out of work and is actively looking for work
    blocked       bool // m is blocked on a note
    inwb          bool // m is executing a write barrier
    newSigstack   bool // minit on C thread called sigaltstack
    printlock     int8
    incgo         bool   // m is executing a cgo call
    freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
    fastrand      [2]uint32
    needextram    bool
    traceback     uint8
    ncgocall      uint64      // number of cgo calls in total
    ncgo          int32       // number of cgo calls currently in progress
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
    nextwaitm     muintptr       // next m waiting for lock
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
    syscall   libcall // stores syscall parameters on windows

    mOS
}
```

抽象数据结构，可以认为是 processor 的抽象，m 必须获得 p 才能执行:

```go
type p struct {
    lock mutex

    id          int32
    status      uint32 // one of pidle/prunning/...
    link        puintptr
    schedtick   uint32     // incremented on every scheduler call
    syscalltick uint32     // incremented on every system call
    sysmontick  sysmontick // last tick observed by sysmon
    m           muintptr   // back-link to associated m (nil if idle)
    mcache      *mcache
    racectx     uintptr

    deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
    deferpoolbuf [5][32]*_defer

    // Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
    goidcache    uint64
    goidcacheend uint64

    // Queue of runnable goroutines. Accessed without lock.
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    // runnext, if non-nil, is a runnable G that was ready'd by
    // the current G and should be run next instead of what's in
    // runq if there's time remaining in the running G's time
    // slice. It will inherit the time left in the current time
    // slice. If a set of goroutines is locked in a
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

    // gcMarkWorkerStartTime is the nanotime() at which this mark
    // worker started.
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
    // accessed atomically. keep at top to ensure alignment on 32-bit systems.
    goidgen  uint64
    lastpoll uint64

    lock mutex

    // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
    // sure to call checkdead().

    midle        muintptr // idle m's waiting for work
    nmidle       int32    // number of idle m's waiting for work
    nmidlelocked int32    // number of locked m's waiting for work
    mnext        int64    // number of m's that have been created and next M ID
    maxmcount    int32    // maximum number of m's allowed (or die)
    nmsys        int32    // number of system m's not counted for deadlock
    nmfreed      int64    // cumulative number of freed m's

    ngsys uint32 // number of system goroutines; updated atomically

    pidle      puintptr // idle p's
    npidle     uint32
    nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

    // Global runnable queue.
    runqhead guintptr
    runqtail guintptr
    runqsize int32

    // Global cache of dead G's.
    gflock       mutex
    gfreeStack   *g
    gfreeNoStack *g
    ngfree       int32

    // Central cache of sudog structs.
    sudoglock  mutex
    sudogcache *sudog

    // Central pool of available defer structs of different sizes.
    deferlock mutex
    deferpool [5]*_defer

    // freem is the list of m's waiting to be freed when their
    // m.exited is set. Linked through m.freelink.
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

    procresizetime int64 // nanotime() of last change to gomaxprocs
    totaltime      int64 // ∫gomaxprocs dt up to procresizetime
}
```

```
                             ┌────┐  ┌────┐              ┌────────┐
                             │ P  │  │ P  │              │ schedt │
                             └────┘  └────┘              └────────┘                  ┌─┬─┬─┬─┬─┬─┬─┐
                                │       │                     │             ┌───────▶│G│G│G│G│G│G│G│
                                │       │                     │             │        └─┴─┴─┴─┴─┴─┴─┘
                                │       │                     └─────────────┘       ┌──────────────────┐
                                │       │                                           │ Global Run Queue │
                   ┌─────┐      ▼       ▼      ┌─────┐                              └──────────────────┘
                   │     │     .─.     .─.     │     │                                        │
                   │     │    ( M )   ( M )    │     │                                        │
        ┌─┬─┬─┬─┬─┬─┬─┐  │     `─'     `─'     │  ┌─┬─┬─┬─┐                                   │
        │G│G│G│G│G│G│G│  │      ▲       ▲      │  │G│G│G│G│                                   │
        └─┴─┴─┴─┴─┴─┴─┘  │      │       │      │  └─┴─┴─┴─┘                                   │
                         └──────┘       └──────┘                                              │
      ┌──────────────────┐                        ┌──────────────────┐                        │
      │ Local Run Queue  │                        │ Local Run Queue  │                        │
      └──────────────────┘                        └──────────────────┘                        │
                                                            ▲                                 │
                                                            │                          ┌──────┴──┐
                                                            └──────────────────────────┤  steal  │
                                                                                       └─────────┘
```



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MzYwMjIwODBdfQ==
-->