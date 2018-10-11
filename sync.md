# sync

## 线性一致性模型

从原理上来讲，atomic 操作和非 atomic 操作之间不满足线性一致性模型。这和现代计算机的 CPU 乱序执行，以及 compiler 为优化而进行的指令重排有关。在 C++ 中针对各种场景和性能需求提供了各种 memory order 选项：

1. memory_order_relaxed    Relaxed operation: 只保证当前操作的原子性，不保证其它原子变量和非原子变量读写的顺序，也不进行任何同步。就是说 CPU 和编译器可以任意重排其它指令。
2. memory_order_consume    和 load 搭配使用时，相当于执行了一个 consume 操作，当前线程依赖 loaded 的值的读写都不能被 reorder 到 load 操作之前。其它线程中对依赖的变量的写操作如果 release 了同一个 atomic 变量，在当前线程中马上可见(release-consume 语义)。
3. memory_order_acquire    只能用在 load 操作中，当前线程中在该 load 之后发生的所有读写，都不能被 reorder 到 load 之前。其它线程中所有写入操作，如果对该 atomic 变量执行了 release 操作，那么其之前的所有写操作在当前线程都看得到(release-acquire 语义)。
4. memory_order_release    只能和 store 操作搭配使用，当前线程中发生在 store 之前的所有读写都不能被 reorder 到 store 操作之后。当前线程在 store 之前发生的所有写操作在其它线程执行同一个 atomic 变量的 load 操作之后，便对其可见(release-acquire 语义)。在 store 时依赖的那些变量(可能为非原子变量)，在其它线程执行 consume 操作之后也会变为可见(release-consume 语义)。
5. memory_order_acq_rel    提供给 read-modify-write 操作用。这种操作会既执行 acquire 又执行 release。当前线程中的读写不能被 reorder 到该操作之前或之后。其它线程中对同一 atomic 变量执行 rlease 操作的写在当前线程中执行 rmw 之前都可见，并且 rmw 操作结果对其它 acquire 相同 atomic 变量的线程也是可见的。
6. memory_order_seq_cst    可以在 load、store 和 rmw 操作中使用。该内存序前提下，和 load 操作搭配使用会执行 acquire 操作，和 store 搭配时会执行 release 操作，和 rmw 搭配会同时执行 acquire 和 release 操作。所有线程观察到的 atomic 变量修改顺序都是一致的。

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

而线性一致的场景下，我们可以用全局事件发生的顺序来推断最终的内存状态。但因为这是最严格的时序，所以硬件同步的成本较高。如果我们的 atomic 变量只用来做全局的简单计数，比如 counter，那么在 Go 中就一定会比 C++ 一类提供了 memory order 选项的语言消耗更多的成本。

但如果 atomic.Load 和 atomic.Store 提供像 C++ 一样的 memory_order 选项，那么又会带给程序员一定的心智负担，所以看起来 Go 官方并不打算提供这样的选项，比如官方组成员是这么说的：

> To be honest, I don't think that people who need to worry about the slow down inherent in using sequential consistency for atomic operations are going to choose Go as their programming language.

关于 atomic 和 memory order 的讨论可以参见: [这里](https://github.com/golang/go/issues/5045)

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

在使用 lock 指令时，~~会导致 CPU 锁总线。~~

订正:

> Actually modern Intel CPUs (since the Pentium pro) only lock the bus in very rare exceptions. Generally they use cache locking which is much, much more efficient and basically just follows from the usual cache coherence protocol (e.g. exclusive state in MESI). 

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
// Mutex 是互斥锁
// 其零值是一个 unlocked 的互斥量
// 在被首次使用之后，Mutex 就不应该发生拷贝动作了
type Mutex struct {
    state int32
    sema  uint32
}
```

加锁过程:

```go
// 对 m 上锁
// 如果锁已经在使用中
// 调用 Lock 的 goroutine 会陷入阻塞
// 直到 mutex 变为可用
func (m *Mutex) Lock() {
    // 当前直接就是已解锁的 mutex
    // 直接用 atomic cas，更快
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }

    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // 这里 if 有 starvation 的判断
        // 在饥饿模式时不能自旋，因为所有权被移交给等待的 goroutine 了
        // 所以我们没办法获得 mutex
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 这里会做积极的自旋
            // 没有其它饥饿的 goroutine 的话，我们尽量直接就设置 mutexWoken flag
            // 这样在 Unlock 的时候就不用唤醒其它被阻塞的 goroutine 了
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                // 设置 mutexWoken flag
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            // 进 runtime 自旋
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old

        // 如果 mutex 处于 starving 状态，就不应该武断地抢锁了
        // 新来的 goroutine 应该先去排队
        if old&mutexStarving == 0 {
            // 说明老状态里没有 starving 那一位
            // 即说明原来的 mutex 不是 starvation 状态
            // 给新的 state 标上 locked 这位
            new |= mutexLocked
        }

        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }

        // 当前 goroutine 将 mutex 切换到 starvation 模式
        // 如果 mutex 当前已经被 unlock 了，就不要做这个切换了
        // Unlock 的时候会认为一个 starving 的 mutex 一定会有等待的 goroutine，
        // 这种情况下一定为 true
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }

        if awoke {
            // 当前 goroutine 是处于 awoke 状态
            // 但是从 mutex 里拿到的状态并没有 mutexWoken 这个 flag
            // 说明这里发生了 bug
            // PS: 这种情况下应该是没有 waiters 的
            // PS: 而是当前的加锁的新 goroutine 直接进入唤醒流程
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }

            // goroutine 被从 sleep 唤醒
            // 所以我们需要在两种情况(starving  和非 starving 的)下
            // :都 reset 掉这个 flag
            new &^= mutexWoken
        }

        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }

            // 如果之前已经等待过了，那么直接插到队列最前面
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            runtime_SemacquireMutex(&m.sema, queueLifo)
            // 如果等待时间超过了阈值，那么就进入 starving 状态
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            if old&mutexStarving != 0 {
                // 如果当前 goroutine 被唤醒，且 mutex 处于 starvation 状态
                // 这时候控制权被移交到给了我们，但 mutex 不知道怎么回事处于不一致的状态:
                // mutexLocked 标识位还没有设置，但我们却仍然认为当前 goroutine 正在等待这个 mutex。说明是个 bug，需要修正
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式
                    // 必须要在这里退出，且考虑等待时间
                    // 饥饿模式很低效，一旦两个 goroutine 同时将 mutex 切换到饥饿模式
                    // 可能会彼此无限地锁下去
                    // ??
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

解锁过程:

```go
// 解锁 m，对未加锁的 mutex 解锁会引起错误
// 被加锁的 mutex 并不是和具体的某个 goroutine 绑定的
// 完全可以在一个 goroutine 中加锁并在另外的 goroutine 中解锁
func (m *Mutex) Unlock() {

    // 干掉 mutexLocked 的标识位
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }

    // 如果新的状态表示 mutex 之前没有处于饥饿状态
    if new&mutexStarving == 0 {
        old := new
        for {
            // 如果当前没有处于饥饿模式等待中的 goroutine，或者当前这个 goroutine 已经
            // 被唤醒或抢到了锁，没有必要再唤醒其它 goroutine 了
            // 饥饿模式中，管理权会直接会被直接从 unlocking goroutine 移交给下一个 waiter
            // 当前 goroutine 并不在这个链条中，
            // 因为我们在 unlock 上面的 mutex 时，没有观察到 mutexStarving 的标识位
            // 所以直接 return 让路
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }

            // 获取唤醒其它人的权力
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式: 将 mutex 的所有权移交给下一个 waiter
        // 注意: mutexLocked 没有设置，waiter 被唤醒后会设置这个标识
        // 但是 mutex 在 waiter 被唤醒后，如果 mutexStarving 位是 1 的话
        // 仍然会被认为是上锁的，所以新来的 goroutine 是没法获取这个锁的
        runtime_Semrelease(&m.sema, true)
    }
}

```


## sync.RWMutex

reader 加解锁过程:

```go
// RWMutex 是 reader/writer 互斥锁
// 这种锁可以被任意数量的 reader 或者单独的一个 writer 所持有
// 其零值是遇 unlocked mutex
//
// RWMutex 在首次使用后就不应该被拷贝了
//
// 如果一个 goroutine 持有了 RWMutex 用来做读操作
// 这时候另一个 goroutine 可能会调用 Lock
// 在这之后，就不会有任何 goroutine 会获得 read lock 了
// 直到最初的 read lock 被释放。
// 需要注意，这种锁是禁止递归的 read locking 的。
// 这是为了保证锁最终一定能够到达可用状态;
// 一个阻塞的 Lock 的调用会排它地阻止其它 readers 获取到这个锁
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}

const rwmutexMaxReaders = 1 << 30

// RLock 锁住 rw 来进行读操作
//
// 不能被使用来做递归的 read locking; 一个阻塞的 Lock 调用会阻止其它新 readers 获取当前锁
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 有 writer 挂起，等待其操作完毕。
        runtime_Semacquire(&rw.readerSem)
    }
}

// RUnlock 相当于 RLock 调用的逆向操作;
// 其不会影响到其它同时持锁的 reader 们
// 如果当前 rw 不是被锁住读的状态，那么就是一个 bug
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxReaders {
            throw("sync: RUnlock of unlocked RWMutex")
        }
        // 有 writer 正在挂起
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            // 最后一个 reader 负责 unblock writer
            runtime_Semrelease(&rw.writerSem, false)
        }
    }
}

```

writer 加解锁过程:

```go
// Lock 对 rw 加写锁
// 如果当前锁已经被锁住进行读或者进行写
// Lock 会阻塞，直到锁可用
func (rw *RWMutex) Lock() {
    // First, resolve competition with other writers.
    // 首先需要解决和其它 writer 进行的竞争，这里是去抢 RWMutex 中的 Mutex 锁
    rw.w.Lock()
    // 抢到了上面的锁之后，通知所有 reader，现在有一个挂起的 writer 等待写入了
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 等待最后的 reader 将其唤醒
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_Semacquire(&rw.writerSem)
    }
}

// Unlock 将 rw 的读锁解锁。如果当前 rw 没有处于锁定读的状态，那么就是 bug
//
// 像 Mutex 一样，一个上锁的 RWMutex 并没有和特定的 goroutine 绑定。
// 可以由一个 goroutine Lock 它，并由其它的 goroutine 解锁
func (rw *RWMutex) Unlock() {

    // 告诉所有 reader 现在没有活跃的 writer 了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
        throw("sync: Unlock of unlocked RWMutex")
    }
    // Unblock 掉所有正在阻塞的 reader
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false)
    }
    // 让其它的 writer 可以继续工作
    rw.w.Unlock()
}

```

## sync.Cond

sync.Cond 本质上就是利用 Mutex 或 RWMutex 的 Lock 会阻塞，来实现了一套事件通知机制。

```go

// Cond 实现了一种条件变量，可以让 goroutine 都等待、或宣布一个事件的发生
//
// 每一个 Cond 都有一个对应的 Locker L，可以是一个 *Mutex 或者 *RWMutex
// 当条件发生变化及调用 Wait 方法时，必须持有该锁
//
// Cond 在首次使用之后同样不能被拷贝
type Cond struct {
    noCopy noCopy

    // 在观测或修改条件时，必须持有 L
    L Locker

    notify  notifyList
    checker copyChecker
}

func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}

// Wait 会原子地解锁 c.L，并挂起当前调用 Wait 的 goroutine
// 之后恢复执行时，Wait 在返回之前对 c.L 加锁。和其它系统不一样
// Wait 在被 Broadcast 或 Signal 唤醒之前，是不能返回的
//
// 因为 c.L 在 Wait 第一次恢复执行之后是没有被锁住的，调用方
// 在 Wait 返回之后没办法假定 condition 为 true。
// 因此，调用方应该在循环中调用 Wait
//
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    .. 这时候 condition 一定为 true..
//    c.L.Unlock()
//
func (c *Cond) Wait() {
    c.checker.check()
    t := runtime_notifyListAdd(&c.notify)
    c.L.Unlock()
    runtime_notifyListWait(&c.notify, t)
    c.L.Lock()
}

// Signal 只唤醒等待在 c 上的一个 goroutine。
// 对于 caller 来说在调用 Signal 时持有 c.L 也是允许的，不过没有必要
func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}

// Broadcast 唤醒所有在 c 上等待的 goroutine
// 同样在调用 Broadcast 时，可以持有 c.L，但没必要
func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify)
}

// 检查结构体是否被拷贝过，因为其持有指向自身的指针
// 指针值和实际地址不一致时，即说明发生了拷贝
type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
        !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
        uintptr(*c) != uintptr(unsafe.Pointer(c)) {
        panic("sync.Cond is copied")
    }
}

// noCopy may be embedded into structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock() {}
```

## sync.Map

数据结构:

```go

// sync.Map 类似 map[interface{}]interface{}，但是是并发安全的。
// Load，Store 和 delete 是通过延迟操作来均摊成本而达到常数时间返回的
//
// sync.Map 类型是为特殊目的准备的，一般的代码还是应该使用普通的 Go map 类型，
// 并自己完成加锁和多线程协作，这样能够有更好的类型安全并且更易维护。
//
// 这里的 Map 是为了两种用例来优化的:
// (1) 当某个指定的 key 只会被写入一次，但是会被读取非常多次，例如像不断增长的 caches。
// (2) 当多个 goroutine 分别分布读、写和覆盖不同的 key。
// 这两种场景下，使用 sync.Map，相比普通的 map 配合 Mutex 或 RWMutex，可以大大降低锁的竞争
//
// Map 的零值是可以使用的空 Map。当 Map 被首次使用之后，就不能再被拷贝了
type Map struct {
    mu Mutex

    // read 包含了 map 内容的一部分，这部分进行并发访问是安全的(无论持有 mu 或未持有)
    //
    // read 字段本身对 load 操作而言是永远安全的，但执行 store 操作时，必须持有 mu
    //
    // 在 read 中存储的 entries 可以在不持有 mu 的情况下进行并发更新，但更新一个之前被 expunged
    // 的项需要持有 mu 的前提下，将该 entry 被拷贝到 dirty map 并将其 unexpunged(就是 expunged 的逆操作)
    read atomic.Value // readOnly

    // dirty 包含了 map 中那些在访问时需要持有 mu 的部分内容
    // 为了确保 dirty map 的元素能够被快速地移动到 read map 中
    // 它也包含了那些 read map 中未删除(non-expunged)的项
    //
    // expunged 掉的 entries 不会在 dirty map 中存储。被 expunged 的 entry，
    // 如果要存新值，需要先执行 expunged 逆操作，然后添加到 dirty map，然后再进行更新
    //
    // 如果 dirty map 为 nil，下一个对 map 的写入会初始化该 map，并对干净 map 进行一次浅拷贝
    // 并忽略那些过期的 entry
    dirty map[interface{}]*entry

    // misses 计算从 read map 上一次被更新开始的需要 lock mu 来进行的 load 次数
    //
    // 一旦发生了足够多的 misses 次数，足以覆盖到拷贝 dirty map 的成本，dirty map 就会被合并进
    // read map(在 unamended 状态下)，并且下一次的 store 操作则会生成一个新的 dirty map
    misses int
}

// readOnly 是原子地存在 Map.read 字段中的不可变结构
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 如果 dirty map 中包含有不在 m 中的项，那么 amended = true
}

// expunged 是一个任意类型的指针，用来标记从 dirty map 中删除的项
var expunged = unsafe.Pointer(new(interface{}))

// entry 是 map 对应一个特定 key 的槽
type entry struct {
    // p 指向 entry 对应的 interface{} 类型的 value
    //
    // 如果 p == nil，那么说明对应的 entry 被删除了，曲 m.dirty == nil
    //
    // 如果 p == expunged，说明 entry 被删除了，但 m.dirty != nil，且该 entry 在 m.dirty 中不存在
    //
    // 除了上述两种情况之外，entry 则是合法的值并且在 m.read.m[key] 中存在
    // 如果 m.dirty != nil，也会在 m.dirty[key] 中
    //
    // 一个 entry 项可以被 atomic cas 替换为 nil 来进行删除: 当 m.dirty 之后被创建的话，
    // 会原子地将 nil 替换为 expunged，且不设置 m.dirty[key] 的值。
    //
    // 一个 entry 对应的值可以用 atomic cas 来更新，前提是 p != expunged。
    // 如果 p == expunged，entry 对应的值只能在首次赋值 m.dirty[key] = e 之后进行
    // 这样查找操作可以用 dirty map 来找到这个 entry
    p unsafe.Pointer // *interface{}
}

```

Load:

```go
func newEntry(i interface{}) *entry {
    return &entry{p: unsafe.Pointer(&i)}
}

// 返回 map 中 key 对应的值，如果不存在，返回 nil
// ok 会返回该值是否在 map 中存在
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        // 当 m.dirty 已被合并到 read 阻塞在 m.mu 时，避免报告不应该报告的 miss (如果未来的
        // 同一个 key 的 loads 不会发生 miss，那么为了这个 key 而拷贝 dirty map 就不值得了)
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 无论 entry 是否存在，都记录一次 miss:
            // 这个 key 会始终走 slow path(加锁的)，直到 dirty map 被合并到 read map
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

func (e *entry) load() (value interface{}, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*interface{})(p), true
}

```

Store:

```go

// 为某一个 key 的 value 赋值
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            // entry 之前被 expunged 了，表明这时候存在非空的 dirty map
            // 且该 entry 不在其中
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        e.storeLocked(&value)
    } else {
        if !read.amended {
            // 为 dirty map 增加第一个新的 key
            // 确保分配内存，并标记 read-only map 为 incomplete(amended = true)
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

// tryStore 在 entry 没有被 expunged 时存储 value
//
// 如果 entry 被 expunged 了，tryStore 会返回 false 并且放弃对 entry 的 value 赋值
func (e *entry) tryStore(i *interface{}) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// unexpungeLocked 确保该 entry 没有被标记为 expunged
//
// 如果 entry 之前被 expunged 了，它必须在 m.mu 被解锁前，被添加到 dirty map
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// storeLocked 无条件地存储对应 entry 的值
//
// entry 必须未被 expunged
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

```

LoadOrStore:

```go
// LoadOrStore 如果 key 对应的值存在，那么就返回
// 否则的话会将参数的 value 存储起来，并返回该值
// loaded 如果为 true，表示值是被加载的，false 则表示实际执行的是 store 操作
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    // 确定命中的话，避免锁
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        actual, loaded, ok := e.tryLoadOrStore(value)
        if ok {
            return actual, loaded
        }
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        actual, loaded, _ = e.tryLoadOrStore(value)
    } else if e, ok := m.dirty[key]; ok {
        actual, loaded, _ = e.tryLoadOrStore(value)
        m.missLocked()
    } else {
        if !read.amended {
            // 为 dirty map 添加第一个新 key
            // 确保分配好内存，并标记 read-only map 为 incomplete
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
        actual, loaded = value, false
    }
    m.mu.Unlock()

    return actual, loaded
}

// tryLoadOrStore 原子地 load 或 store 一个 entry 对应的 value
// 前提是该 entry 没有被 expunged
//
// 如果 entry 被 expunged 的话，tryLoadOrStore 会不做任何修改，并返回 ok==false
func (e *entry) tryLoadOrStore(i interface{}) (actual interface{}, loaded, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return nil, false, false
    }
    if p != nil {
        return *(*interface{})(p), true, true
    }

    // 首次 load 之后拷贝 interface，以使该方法对逃逸分析更顺从: 如果我们触发了 "load" 路径
    // 或者 entry 被 expunged 了，我们不应该造成 heap-allocating
    ic := i
    for {
        if atomic.CompareAndSwapPointer(&e.p, nil, unsafe.Pointer(&ic)) {
            return i, false, true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return nil, false, false
        }
        if p != nil {
            return *(*interface{})(p), true, true
        }
    }
}

```

Delete:

```go
// Delete 删除 key 对应的 value
func (m *Map) Delete(key interface{}) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    if ok {
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}

```

Range 遍历:

```go

// Range 按每个 key 和 value 在 map 里出现的顺序调用 f
// 如果 f 返回 false，range 会停止迭代
//
// Range 不需要严格地对应 Map 内容的某个快照: 就是说，每个 key 只会被访问一次，
// 但如果存在某个 key 被并发地更新或者删除了，Range 可以任意地返回修改前或修改后的值
//
// Range 不管 Map 中有多少元素都是 O(N) 的时间复杂度
// 即使 f 在一定数量的调用之后返回 false 也一样
func (m *Map) Range(f func(key, value interface{}) bool) {
    // 需要在一开始调用 Range 的时候，就能够迭代所有已经在 map 里的 key
    // 如果 read.amended 是 false，那么 read.m 就可以满足要求了
    // 而不需要我们长时间持有 m.mu
    read, _ := m.read.Load().(readOnly)
    if read.amended {
        // m.dirty 包含有 未出现在 read.m 中的 key。幸运的是 Range 已经是 O(N) 了
        // (假设 caller 不会中途打断)，所以对于 Range 的调用必然会分阶段完整地拷贝整个 map:
        // 这时候我们可以直接把 dirty map 拷贝到 read!
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        if read.amended {
            read = readOnly{m: m.dirty}
            m.read.Store(read)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }

    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue
        }
        if !f(k, v) {
            break
        }
    }
}

```

其它小函数:

```go

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```

# 参考资料

http://www.weixianmanbu.com/article/736.html

https://www.cnblogs.com/gaochundong/p/lock_free_programming.html

https://en.cppreference.com/w/cpp/atomic/memory_order

https://github.com/brpc/brpc/blob/master/docs/cn/atomic_instructions.md
