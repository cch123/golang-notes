# lock

## atomic

```go
TEXT ·AddUint32(SB),NOSPLIT,$0-20
    MOVQ    addr+0(FP), BP
    MOVL    delta+8(FP), AX
    MOVL    AX, CX
    LOCK
    XADDL   AX, 0(BP)
    ADDL    AX, CX
    MOVL    CX, new+16(FP)
    RET
```

```go
0x0036 00054 (atomic.go:10)    MOVL    $10, CX
0x003b 00059 (atomic.go:10)    LOCK
0x003c 00060 (atomic.go:10)    XADDL    CX, (AX)
```

在 intel 平台上被翻译为:

```shell
mov ecx, 0xa
lock xadd DWORD PTR [rax], ecx
```

lock 指令前缀可以使许多指令操作（ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG）变成原子操作。CMPXCHG 指令用来实现 CAS 操作。

atomic.CompareAndSwap 即是使用 lock cmpxchg 来实现的。

在使用 lock 指令时，会导致 CPU 锁总线。

## waitgroup

```go
// 在主 goroutine 中 Add 和 Wait，在其它 goroutine 中 Done
// 在第一次使用之后，不能对 WaitGroup 再进行拷贝
type WaitGroup struct {
    noCopy noCopy

    // state1 的高 32 位是计数器，低 32 位是 waiter 计数
    // 64 位的 atomic 操作需要按 64 位对齐，但是 32 位编译器没法保证这种对齐
    // 所以分配 12 个字节(多分配了 4 个字节)，以使我们使用的位都在对齐的位置
    // 0000...0000      0000...0000       0000...0000
    // |- 4 bytes-|    |- 4 bytes -|     |- 4 bytes -|
    //     使用             不使用             使用
    state1 [12]byte
    sema   uint32
}

func (wg *WaitGroup) state() *uint64 {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        return (*uint64)(unsafe.Pointer(&wg.state1))
    } else {
        return (*uint64)(unsafe.Pointer(&wg.state1[4]))
    }
}

// Add 一个 delta，delta 可能是负值，在 WaitGroup 的 counter 上增加该值
// 如果 counter 变成 0，所有阻塞在 Wait 函数上的 goroutine 都会被释放
// 如果 counter 变成了负数，Add 会直接 panic
// 当 counter 是 0 且 Add 的 delta 为正的操作必须发生在 Wait 调用之前。
// 而当 counter > 0 且 Add 的 delta 为负的操作则可以发生在任意时刻。

// Typically this means the calls to Add should execute before the statement
// creating the goroutine or other event to be waited for.
// If a WaitGroup is reused to wait for several independent sets of events,
// new Add calls must happen after all previous Wait calls have returned.
// See the WaitGroup example.
func (wg *WaitGroup) Add(delta int) {
    statep := wg.state()

    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32)
    w := uint32(state)

    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }
    // This goroutine has set counter to 0 when waiters > 0.
    // Now there can't be concurrent mutations of state:
    // - Adds must not happen concurrently with Wait,
    // - Wait does not increment waiters if it sees counter == 0.
    // Still do a cheap sanity check to detect WaitGroup misuse.
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // Reset waiters count to 0.
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(&wg.sema, false)
    }
}

// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}

// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
    statep := wg.state()

    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)
        w := uint32(state)
        if v == 0 {
            // Counter is 0, no need to wait.
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }

        // Increment waiters count.
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            runtime_Semacquire(&wg.sema)
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

## futex

```go
// This implementation depends on OS-specific implementations of
//
//    futexsleep(addr *uint32, val uint32, ns int64)
//        Atomically,
//            if *addr == val { sleep }
//        Might be woken up spuriously; that's allowed.
//        Don't sleep longer than ns; ns < 0 means forever.
//
//    futexwakeup(addr *uint32, cnt uint32)
//        If any procs are sleeping on addr, wake up at most cnt.

const (
    mutex_unlocked = 0
    mutex_locked   = 1
    mutex_sleeping = 2

    active_spin     = 4
    active_spin_cnt = 30
    passive_spin    = 1
)

// Possible lock states are mutex_unlocked, mutex_locked and mutex_sleeping.
// mutex_sleeping means that there is presumably at least one sleeping thread.
// Note that there can be spinning threads during all states - they do not
// affect mutex's state.

// We use the uintptr mutex.key and note.key as a uint32.
//go:nosplit
func key32(p *uintptr) *uint32 {
    return (*uint32)(unsafe.Pointer(p))
}

func lock(l *mutex) {
    gp := getg()

    if gp.m.locks < 0 {
        throw("runtime·lock: lock count")
    }
    gp.m.locks++

    // Speculative grab for lock.
    v := atomic.Xchg(key32(&l.key), mutex_locked)
    if v == mutex_unlocked {
        return
    }

    // wait is either MUTEX_LOCKED or MUTEX_SLEEPING
    // depending on whether there is a thread sleeping
    // on this mutex. If we ever change l->key from
    // MUTEX_SLEEPING to some other value, we must be
    // careful to change it back to MUTEX_SLEEPING before
    // returning, to ensure that the sleeping thread gets
    // its wakeup call.
    wait := v

    // On uniprocessors, no point spinning.
    // On multiprocessors, spin for ACTIVE_SPIN attempts.
    spin := 0
    if ncpu > 1 {
        spin = active_spin
    }
    for {
        // Try for lock, spinning.
        for i := 0; i < spin; i++ {
            for l.key == mutex_unlocked {
                if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
                    return
                }
            }
            procyield(active_spin_cnt)
        }

        // Try for lock, rescheduling.
        for i := 0; i < passive_spin; i++ {
            for l.key == mutex_unlocked {
                if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
                    return
                }
            }
            osyield()
        }

        // Sleep.
        v = atomic.Xchg(key32(&l.key), mutex_sleeping)
        if v == mutex_unlocked {
            return
        }
        wait = mutex_sleeping
        futexsleep(key32(&l.key), mutex_sleeping, -1)
    }
}

func unlock(l *mutex) {
    v := atomic.Xchg(key32(&l.key), mutex_unlocked)
    if v == mutex_unlocked {
        throw("unlock of unlocked lock")
    }
    if v == mutex_sleeping {
        futexwakeup(key32(&l.key), 1)
    }

    gp := getg()
    gp.m.locks--
    if gp.m.locks < 0 {
        throw("runtime·unlock: lock count")
    }
    if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
        gp.stackguard0 = stackPreempt
    }
}

// One-time notifications.
func noteclear(n *note) {
    n.key = 0
}

func notewakeup(n *note) {
    old := atomic.Xchg(key32(&n.key), 1)
    if old != 0 {
        print("notewakeup - double wakeup (", old, ")\n")
        throw("notewakeup - double wakeup")
    }
    futexwakeup(key32(&n.key), 1)
}

func notesleep(n *note) {
    gp := getg()
    if gp != gp.m.g0 {
        throw("notesleep not on g0")
    }
    ns := int64(-1)
    if *cgo_yield != nil {
        // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
        ns = 10e6
    }
    for atomic.Load(key32(&n.key)) == 0 {
        gp.m.blocked = true
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
    }
}

// May run with m.p==nil if called from notetsleep, so write barriers
// are not allowed.
//
//go:nosplit
//go:nowritebarrier
func notetsleep_internal(n *note, ns int64) bool {
    gp := getg()

    if ns < 0 {
        if *cgo_yield != nil {
            // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
            ns = 10e6
        }
        for atomic.Load(key32(&n.key)) == 0 {
            gp.m.blocked = true
            futexsleep(key32(&n.key), 0, ns)
            if *cgo_yield != nil {
                asmcgocall(*cgo_yield, nil)
            }
            gp.m.blocked = false
        }
        return true
    }

    if atomic.Load(key32(&n.key)) != 0 {
        return true
    }

    deadline := nanotime() + ns
    for {
        if *cgo_yield != nil && ns > 10e6 {
            ns = 10e6
        }
        gp.m.blocked = true
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
        if atomic.Load(key32(&n.key)) != 0 {
            break
        }
        now := nanotime()
        if now >= deadline {
            break
        }
        ns = deadline - now
    }
    return atomic.Load(key32(&n.key)) != 0
}

func notetsleep(n *note, ns int64) bool {
    gp := getg()
    if gp != gp.m.g0 && gp.m.preemptoff != "" {
        throw("notetsleep not on g0")
    }

    return notetsleep_internal(n, ns)
}

// same as runtime·notetsleep, but called on user g (not g0)
// calls only nosplit functions between entersyscallblock/exitsyscall
func notetsleepg(n *note, ns int64) bool {
    gp := getg()
    if gp == gp.m.g0 {
        throw("notetsleepg on g0")
    }

    entersyscallblock()
    ok := notetsleep_internal(n, ns)
    exitsyscall()
    return ok
}

func pauseSchedulerUntilCallback() bool {
    return false
}

func checkTimeouts() {}
```

## sync.RWMutex


## sync.Map

# 参考资料

http://www.weixianmanbu.com/article/736.html

https://www.cnblogs.com/gaochundong/p/lock_free_programming.html
