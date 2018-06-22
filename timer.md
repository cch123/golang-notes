# 定时器

## 简单用法

什么 Date，Parse，ParseInLocation，就不说了。主要关注下面几个函数：

ticker 相关:

```go
func Tick(d Duration) <-chan Time
func NewTicker(d Duration) *Ticker
func (t *Ticker) Stop()
```

timer 相关:

```go
func After(d Duration) <-chan Time
func NewTimer(d Duration) *Timer
func (t *Timer) Reset(d Duration) bool
func (t *Timer) Stop() bool
```

## 源码分析

### 数据结构

```
                                          ┌────────┐                                                                                                     
                                          │ timers │                                                                                                     
                                          ├────┬───┴┬────┬────┬────┬────┬────┬───────────────────────┬────┐                                              
                                          │    │    │    │    │    │    │    │                       │    │                                              
                                          │  0 │  1 │  2 │  3 │  4 │  5 │  6 │            ...        │ 63 │                                              
                                          └────┴────┴────┴────┴────┴────┴────┴───────────────────────┴────┘                                              
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │ │                                          │            │    │                                          │
                                              │ │          ┌────────────────────┐          │            │    │          ┌────────────────────┐          │
                                              │ │────────▶ │ cacheline size * N │ ◀────────┤            │    │────────▶ │ cacheline size * N │ ◀────────┤
                                              │ │          └────────────────────┘          │            │    │          └────────────────────┘          │
                                              │ ├─────────────┬─────────────────┬──────────┤            │    ├─────────────┬─────────────────┬──────────┤
                                              └▶│timersBucket │                 │          │            └───▶│timersBucket │                 │          │
                                                ├─────────────┴─────────────────┤   pad    │                 ├─────────────┴─────────────────┤   pad    │
                                                │          lock mutex           │          │                 │          lock mutex           │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │             gp *g             │          │                 │             gp *g             │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │         created bool          │          │                 │         created bool          │          │
                                                ├───────────────────────────────┤          │    ........     ├───────────────────────────────┤          │
                                                │         sleeping bool         │          │                 │         sleeping bool         │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │       rescheduling bool       │          │                 │       rescheduling bool       │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │       sleepUntil int64        │          │                 │       sleepUntil int64        │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │         waitnote note         │          │                 │         waitnote note         │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │          t []*timer           │          │                 │          t []*timer           │          │
                                                └───────────────────────────────┴──────────┘                 └───────────────────────────────┴──────────┘
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                ▼                                                                                        
                                                              ┌───┐                                                                                      
                                                              │ 0 │                                                                                      
                                                              └───┘                                                                                      
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                ▼                                                                                        
                                                        ┌───┬───┬───┬───┐                                                                                
                                                        │ 1 │ 2 │ 3 │ 4 │                                                                                
                                                        └─┬─┴─┬─┴─┬─┴─┬─┘                                                                                
                        ┌─────────────────────────────────┘   │   │   └──────────────────────────────────────────┐                                       
                        │                              ┌──────┘   └────────┐                                     │                                       
                        ▼                              │                   │                                     ▼                                       
                ┌───┬───┬───┬───┐                      ▼                   ▼                             ┌───┬───┬───┬───┐                               
                │   │   │   │   │              ┌───┬───┬───┬───┐   ┌───┬───┬───┬───┐                     │   │   │   │   │                               
                └─┬─┴─┬─┴───┴───┘              │   │   │   │   │   │   │   │   │   │                     └───┴───┴─┬─┴─┬─┘                               
                  │   │                        └───┴───┴───┴───┘   └───┴───┴───┴───┘                               │   │                                 
        ┌─────────┘   └─────────┐                                                                        ┌─────────┘   └─────┐                           
        │                       │                                                                        │                   │                           
        │                       │                                                                        │                   │                           
        ▼                       ▼                                                                        ▼                   ▼                           
┌───┬───┬───┬───┐       ┌───┬───┬───┬───┐                                                        ┌───┬───┬───┬───┐   ┌───┬───┬───┬───┐                   
│   │   │   │   │       │   │   │   │   │                  .................                     │   │   │   │   │   │   │   │   │   │                   
└───┴───┴───┴───┘       └───┴───┴───┴───┘                                                        └───┴───┴───┴───┘   └───┴───┴───┴───┘                   
```

下面的结论都可以结合上面的图来看。

在 `runtime/time.go` 中定义了 timers 数组:

```go
var timers [timersLen]struct {
    timersBucket
    pad [sys.CacheLineSize - unsafe.Sizeof(timersBucket{})%sys.CacheLineSize]byte
}
```

在 Go 的早期实现中是全局一个 timer 的，但操作全局的 timer 堆要加锁，所以多核心会暴露出因为争锁而性能低下的问题。从某个版本(嗯，我也不知道哪个，大概是 1.10)起，Go 的 timers 修改成了这种多个时间堆的实现方式，目前在 runtime 里写死为 64:

```go
const timersLen = 64
```

嗯，官方表示如果和 GOMAXPROCS 相等的话，那么会在 procresize 的时候重新分配和修改这些时间堆。写死成 64 是内存使用和性能上的一个折衷。如果 GOMAXPROCS 比 64 大的话，那么可能多个 P 会公用同一个时间堆。当然，实际场景中我还没有见过 64 核以上的 CPU。

timers 数组的元素是一个匿名 struct，包含 timersBucket 和 pad 两个成员，这个 pad 是为了填充 struct 到 cacheline 的整数倍，以避免在不同的 P 之间发生 false sharing。在多核心的编程场景中较为常见。timerBucket 的结构:

```go
//go:notinheap
type timersBucket struct {
    lock         mutex
    gp           *g
    created      bool
    sleeping     bool
    rescheduling bool
    sleepUntil   int64
    waitnote     note
    t            []*timer
}
```

其中的 t 就是我们的时间堆了，不过这个和我们传统的 heap 结构稍微有所不同，是分四个叉的，这种设计第一次见。这里的 timersBucket 还有个特殊的注释 `go:notinheap`，官方的说明:

> go:notinheap applies to type declarations. It indicates that a type must never be allocated from the GC'd heap. Specifically, pointers to this type must always fail the runtime.inheap check. The type may be used for global variables, for stack variables, or for objects in unmanaged memory (e.g., allocated with sysAlloc, persistentalloc, fixalloc, or from a manually-managed span). Specifically:
> 1. new(T), make([]T), append([]T, ...) and implicit heap allocation of T are disallowed. (Though implicit allocations are disallowed in the runtime anyway.)
> 2. A pointer to a regular type (other than unsafe.Pointer) cannot be converted to a pointer to a go:notinheap type, even if they have the same underlying type.
> 3. Any type that contains a go:notinheap type is itself go:notinheap. Structs and arrays are go:notinheap if their elements are. Maps and channels of go:notinheap types are disallowed. To keep things explicit, any type declaration where the type is implicitly go:notinheap must be explicitly marked go:notinheap as well.
> 4. Write barriers on pointers to go:notinheap types can be omitted.
> The last point is the real benefit of go:notinheap. The runtime uses it for low-level internal structures to avoid memory barriers in the scheduler and the memory allocator where they are illegal or simply inefficient. This mechanism is reasonably safe and does not compromise the readability of the runtime.

嗯，了解一下就行了，在用户代码中基本不会用得到。

### 时间堆插入

```go
// 分配 timerBucket
// 加锁，添加 timer 进时间堆
func addtimer(t *timer) {
    tb := t.assignBucket()
    lock(&tb.lock)
    tb.addtimerLocked(t)
    unlock(&tb.lock)
}

// 太简单了，就是用 g.m.p 的 id 模 64
// 然后分配对应的 timerBucket
func (t *timer) assignBucket() *timersBucket {
    id := uint8(getg().m.p.ptr().id) % timersLen
    t.tb = &timers[id].timersBucket
    return t.tb
}

// 向时间堆中添加一个 timer，如果时间堆第一次被初始化或者当前的 timer 比之前所有的 timers 都要早，那么就启动(首次初始化)或唤醒(最早的 timer) timerproc
// 函数内假设外部已经对 timers 数组加锁了
func (tb *timersBucket) addtimerLocked(t *timer) {
    // when 必须大于 0，否则会在计算 delta 的时候溢出并导致其它的 runtime timer 永远没法过期
    if t.when < 0 {
        t.when = 1<<63 - 1
    }
    t.i = len(tb.t)
    tb.t = append(tb.t, t)
    siftupTimer(tb.t, t.i)
    if t.i == 0 {
        // 新插入的 timer 比之前所有的都要早
        // 向上调整堆
        if tb.sleeping {
            // 修改 timerBucket 的 sleep 状态
            tb.sleeping = false
            // 唤醒 timerproc
            // 使 timerproc 中的 for 循环不再阻塞在 notesleepg 上
            notewakeup(&tb.waitnote)
        }
        // 在一个 P 上的所有 timers 都在
        // timerproc 中被弹出之后
        // 该 rescheduling 会被标记为 true
        // 并且启动 timerproc 的 goroutine 会被 goparkunlock
        if tb.rescheduling {
            // 该标记会在这里和 timejumpLocked 中被设置为 false
            tb.rescheduling = false
            goready(tb.gp, 0)
        }
    }
    // 如果 timerBucket 是第一次创建，需要启动一个 goroutine
    // 来循环弹出时间堆，内部会根据需要最早触发的 timer
    // 进行相应时间的 sleep
    if !tb.created {
        tb.created = true
        go timerproc(tb)
    }
}
```

插入 timer 到堆中的时候的逻辑是先追加到数组末尾(append)，然后向上 adjust(sift) heap 以重新恢复四叉小顶堆性质。

### 时间堆删除

### time.After

```go
func After(d Duration) <-chan Time {
    return NewTimer(d).C
}

// NewTimer creates a new Timer that will send
// the current time on its channel after at least duration d.
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime,
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}

func startTimer(*runtimeTimer)
```

这个 startTimer 的实现是在 `runtime/time.go` 里:

```go
// startTimer adds t to the timer heap.
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
    addtimer(t)
}

func (t *timer) assignBucket() *timersBucket {
    id := uint8(getg().m.p.ptr().id) % timersLen
    t.tb = &timers[id].timersBucket
    return t.tb
}

// Timerproc runs the time-driven events.
// It sleeps until the next event in the tb heap.
// If addtimer inserts a new earlier event, it wakes timerproc early.
func timerproc(tb *timersBucket) {
    tb.gp = getg()
    for {
        lock(&tb.lock)
        tb.sleeping = false
        now := nanotime()
        delta := int64(-1)
        for {
            if len(tb.t) == 0 {
                delta = -1
                break
            }
            t := tb.t[0]
            delta = t.when - now
            if delta > 0 {
                break
            }
            if t.period > 0 {
                // leave in heap but adjust next time to fire
                t.when += t.period * (1 + -delta/t.period)
                siftdownTimer(tb.t, 0)
            } else {
                // remove from heap
                last := len(tb.t) - 1
                if last > 0 {
                    tb.t[0] = tb.t[last]
                    tb.t[0].i = 0
                }
                tb.t[last] = nil
                tb.t = tb.t[:last]
                if last > 0 {
                    siftdownTimer(tb.t, 0)
                }
                t.i = -1 // mark as removed
            }
            f := t.f
            arg := t.arg
            seq := t.seq
            unlock(&tb.lock)
            if raceenabled {
                raceacquire(unsafe.Pointer(t))
            }
            f(arg, seq)
            lock(&tb.lock)
        }
        if delta < 0 || faketime > 0 {
            // No timers left - put goroutine to sleep.
            tb.rescheduling = true
            goparkunlock(&tb.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
            continue
        }
        // At least one timer pending. Sleep until then.
        tb.sleeping = true
        tb.sleepUntil = now + delta
        noteclear(&tb.waitnote)
        unlock(&tb.lock)
        notetsleepg(&tb.waitnote, delta)
    }
}
```


TODO timejump