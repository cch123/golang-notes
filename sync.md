# sync

## 线性一致性模型

从原理上来讲，atomic 操作和非 atomic 操作之间不满足线性一致性模型。这和现代计算机的 CPU 乱序执行，以及 compiler 为优化而进行的指令重排有关。在 C++ 中针对各种场景和性能需求提供了各种 memory order 选项：

1. memory_order_relaxed    Relaxed operation: 只保证当前操作的原子性，不保证其它读写的顺序，也不进行任何多余的同步。就是说 CPU 和编译器可以任意重排其它指令。
2. memory_order_consume    和 load 搭配使用时，相当于执行了一个 consume 操作，当前线程依赖 loaded 的值的读写都不能被 reorder 到 load 操作之前。其它线程中对依赖的变量的写操作如果 release 了同一个 atomic 变量，在当前线程中马上可见。
3. memory_order_acquire    只能用在 load 操作中，当前线程中在该 load 之后发生的所有读写，都不能被 reorder 到 load 之前。其它线程中所有写入操作，如果对该 atomic 变量执行了 release 操作，那么其之前的所有写操作在当前线程都看得到。
4. memory_order_release    只能用在 store 操作中，当前线程中发生在 store 之前的所有读写都不能被 reorder 到 store 操作之后。当前线程在 store 之前发生的所有写操作在其它线程执行同一个 atomic 变量的获取操作之后便都是可见的了。所有对原子变量的写入都会在 consume 相同原子变量的线程中可见。
5. memory_order_acq_rel    提供给 read-modify-write 操作用。这种操作会既执行 acquire 又执行 release。当前线程中的读写不能被 reorder 到该操作之前或之后。其它线程中对同一 atomic 变量执行 rlease 操作的写在当前线程中执行 rmw 之前都可见，并且 rmw 操作结果对其它 acquire 相同 atomic 变量的线程也是可见的。
6. memory_order_seq_cst    可以在 load、store 和 rmw 操作中使用。load 操作使用时，相当于执行了 acquire，store 相当于执行了 release，rmw 相当于执行了 acquire 和 release。所有线程间观察到的修改顺序都是一致的。

这里面时序最为严格的是 memory_order_seq_cst，这就是我们常说的“线性一致性”。Go 语言的 atomic 类似这个最严格的时序。简单说明即：

1. 当一个 goroutine 中对某个值进行 atomic.Store，在另一个 goroutine 中对同一个变量进行 atomic.Load，那么 Load 之后可以看到 Store 的结果，且可以看到 Store 之前的其它内存写入操作(在 C++ 的文档中可能被称为 side effect)。
2. atomic.Store 全局有序，即你在任何一个 goroutine 中观察到的全局 atomic 变量们的变化顺序一定是一致的，不会出现有违逻辑顺序的出现次序。这个有一些难理解，看一下下面这个 C++ 的例子：

```c++
#include <thread>
#include <atomic>
#include <cassert>
 
std::atomic<bool> x = {false};
std::atomic<bool> y = {false};
std::atomic<int> z = {0};
 
void write_x()
{
    x.store(true, std::memory_order_seq_cst);
}
 
void write_y()
{
    y.store(true, std::memory_order_seq_cst);
}
 
void read_x_then_y()
{
    while (!x.load(std::memory_order_seq_cst))
        ;
    if (y.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
 
void read_y_then_x()
{
    while (!y.load(std::memory_order_seq_cst))
        ;
    if (x.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
 
int main()
{
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join(); b.join(); c.join(); d.join();
    assert(z.load() != 0);  // will never happen
}
```

在非线性一致的场景下，可能会出现线程 c 和线程 d 观察到的 x，y 值分别为 c: true, false; d: false, true。从而导致最终 z 的结果为 0。

而线性一致的场景下，我们可以用全局事件发生的顺序来推断最终的内存状态。但因为这是最严格的时序，所以 compiler 和硬件同步的成本较高。如果我们的 atomic 变量只用来做全局的简单计数，比如 counter，那么在 Go 中就一定会比 C++ 一类提供了 memory order 选项的语言消耗更多的成本。

但如果 atomic.Load 和 atomic.Store 提供像 C++ 一样的 memory_order 选项，那么又会带给程序员一定的心智负担，所以看起来 Go 官方并不打算提供这样的选项。

## atomic 的实现

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
    // 所以分配 12 个字节(多分配了 4 个字节)
    // 当 state 没有按 8 对齐时，我们可以偏 4 个字节来使用
    // 按 8 对齐时：
    // 0000...0000      0000...0000       0000...0000
    // |- 4 bytes-|    |- 4 bytes -|     |- 4 bytes -|
    //     使用              使用             不使用
    // 没有按 8 对齐时：
    // |- 4 bytes-|    |- 4 bytes -|     |- 4 bytes -|
    //    不使用              使用             使用
    // |-low->  ---------> ------> -----------> high-|
    state1 [12]byte
    sema   uint32
}

func (wg *WaitGroup) state() *uint64 {
    // 判断 state 是否按照 8 字节对齐
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 已对齐时，使用低 8 字节即可
        return (*uint64)(unsafe.Pointer(&wg.state1))
    } else {
        // 未对齐时，使用高 8 字节
        return (*uint64)(unsafe.Pointer(&wg.state1))
        return (*uint64)(unsafe.Pointer(&wg.state1[4]))
    }
}

// Add 一个 delta，delta 可能是负值，在 WaitGroup 的 counter 上增加该值
// 如果 counter 变成 0，所有阻塞在 Wait 函数上的 goroutine 都会被释放
// 如果 counter 变成了负数，Add 会直接 panic
// 当 counter 是 0 且 Add 的 delta 为正的操作必须发生在 Wait 调用之前。
// 而当 counter > 0 且 Add 的 delta 为负的操作则可以发生在任意时刻。
// 一般来讲，Add 操作应该在创建 goroutine 或者其它需要等待的事件发生之前调用
// 如果 wg 被用来等待几组独立的事件集合
// 新的 Add 调用应该在所有 Wait 调用返回之后再调用
// 参见 wg 的 example
func (wg *WaitGroup) Add(delta int) {
    statep := wg.state()

    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32) // counter 高位 4 字节
    w := uint32(state) // waiter counter，截断，取低位 4 个字节

    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }

    // 当前 goroutine 已经把 counter 设为 0，且 waiter 数 > 0
    // 这时候不能有状态的跳变
    // - Add 不能和 Wait 进行并发调用
    // - Wait 如果发现 counter 已经等于 0，则不应该对 waiter 数加一了
    // 这里是对 wg 误用的简单检测
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }

    // 重置 waiter 计数为 0
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(&wg.sema, false)
    }
}

// Done 其实就是 wg 的 counter - 1
// 进入 Add 函数后
// 如果 counter 变为 0 会触发 runtime_Semrelease 通知所有阻塞在 Wait 上的 g
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}

// Wait 会阻塞直到 wg 的 counter 变为 0
func (wg *WaitGroup) Wait() {
    statep := wg.state()

    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32) // counter
        w := uint32(state) // waiter count
        if v == 0 { // counter
            return
        }

        // 如果没成功，可能有并发，循环再来一次相同流程
        // 成功直接返回
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            runtime_Semacquire(&wg.sema) // 和上面的 Add 里的 runtime_Semrelease 是对应的
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

## once

```go
// 内含一个锁和用来做原子操作的变量
type Once struct {
    m    Mutex
    done uint32
}

// Do 被用来执行那些只能执行一次的初始化操作
//     config.once.Do(func() { config.init(filename) })
//
// 因为对 Do 的调用直到其中的一个 f 执行之后才会返回，所以
// f 中不能调用同一个 once 实例的 Do 函数，否则会死锁
// 如果 f 内 panic 了，Do 也认为已经返回了，未来对 Do 的调用不会再执行 f

// once.Do(f) 被调用多次时，只有第一次调用会真正的执行 f
// 对于每一个要执行的 f，都需要一个对应的 once 实例
// 在 done 已经被改成 1 之后
// 所有进入函数调用的行为会用 atomic 读取值之后直接返回
func (o *Once) Do(f func()) {
    // 轻量级的原子变量 load
    if atomic.LoadUint32(&o.done) == 1 {
        // 如果原子 load 后发现已经是 1 了，直接返回
        return
    }

    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    // 在 atomic load 的时候为 0，不代表进入 lock 之后也是 0
    // 所以还需要再判断一次
    // 临界区内的判断和修改是比较稳妥的
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

once.Do 实际上是一种优化，只要过程被执行过了，那么之后所有判断都走 atomic，不用进入临界区。

## Mutex

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
    state int32
    sema  uint32
}
```

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }

    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway.
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case.
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            runtime_SemacquireMutex(&m.sema, queueLifo)
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            if old&mutexStarving != 0 {
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    // Exit starvation mode.
                    // Critical to do it here and consider wait time.
                    // Starvation mode is so inefficient, that two goroutines
                    // can go lock-step infinitely once they switch mutex
                    // to starvation mode.
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }
}
```

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {

    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 {
        old := new
        for {
            // If there are no waiters or a goroutine has already
            // been woken or grabbed the lock, no need to wake anyone.
            // In starvation mode ownership is directly handed off from unlocking
            // goroutine to the next waiter. We are not part of this chain,
            // since we did not observe mutexStarving when we unlocked the mutex above.
            // So get off the way.
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // Grab the right to wake someone.
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // Starving mode: handoff mutex ownership to the next waiter.
        // Note: mutexLocked is not set, the waiter will set it after wakeup.
        // But mutex is still considered locked if mutexStarving is set,
        // so new coming goroutines won't acquire it.
        runtime_Semrelease(&m.sema, true)
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

```go
// A RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers or a single writer.
// The zero value for a RWMutex is an unlocked mutex.
//
// A RWMutex must not be copied after first use.
//
// If a goroutine holds a RWMutex for reading and another goroutine might
// call Lock, no goroutine should expect to be able to acquire a read lock
// until the initial read lock is released. In particular, this prohibits
// recursive read locking. This is to ensure that the lock eventually becomes
// available; a blocked Lock call excludes new readers from acquiring the
// lock.
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}

const rwmutexMaxReaders = 1 << 30

// RLock locks rw for reading.
//
// It should not be used for recursive read locking; a blocked Lock
// call excludes new readers from acquiring the lock. See the
// documentation on the RWMutex type.
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // A writer is pending, wait for it.
        runtime_Semacquire(&rw.readerSem)
    }
}

// RUnlock undoes a single RLock call;
// it does not affect other simultaneous readers.
// It is a run-time error if rw is not locked for reading
// on entry to RUnlock.
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxReaders {
            throw("sync: RUnlock of unlocked RWMutex")
        }
        // A writer is pending.
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            // The last reader unblocks the writer.
            runtime_Semrelease(&rw.writerSem, false)
        }
    }
}

// Lock locks rw for writing.
// If the lock is already locked for reading or writing,
// Lock blocks until the lock is available.
func (rw *RWMutex) Lock() {
    // First, resolve competition with other writers.
    rw.w.Lock()
    // Announce to readers there is a pending writer.
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // Wait for active readers.
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_Semacquire(&rw.writerSem)
    }
}

// Unlock unlocks rw for writing. It is a run-time error if rw is
// not locked for writing on entry to Unlock.
//
// As with Mutexes, a locked RWMutex is not associated with a particular
// goroutine. One goroutine may RLock (Lock) a RWMutex and then
// arrange for another goroutine to RUnlock (Unlock) it.
func (rw *RWMutex) Unlock() {

    // Announce to readers there is no active writer.
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
        throw("sync: Unlock of unlocked RWMutex")
    }
    // Unblock blocked readers, if any.
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false)
    }
    // Allow other writers to proceed.
    rw.w.Unlock()
}

```

## sync.Map

# 参考资料

http://www.weixianmanbu.com/article/736.html

https://www.cnblogs.com/gaochundong/p/lock_free_programming.html
