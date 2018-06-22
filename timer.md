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

在 Go 的早期实现中是全局一个 timer 的，但操作全局的 timer 堆要加锁，所有多核心会暴露出因为争锁而性能低下的问题。从某个版本(嗯，我也不知道哪个，大概是 1.10)起，Go 的 timers 修改成了这种多个时间堆的实现方式，目前在 runtime 里写死为 64:

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

```doc
go:notinheap applies to type declarations. It indicates that a type must never be allocated from the GC'd heap. Specifically, pointers to this type must always fail the runtime.inheap check. The type may be used for global variables, for stack variables, or for objects in unmanaged memory (e.g., allocated with sysAlloc, persistentalloc, fixalloc, or from a manually-managed span). Specifically:

1. new(T), make([]T), append([]T, ...) and implicit heap allocation of T are disallowed. (Though implicit allocations are disallowed in the runtime anyway.)

2. A pointer to a regular type (other than unsafe.Pointer) cannot be converted to a pointer to a go:notinheap type, even if they have the same underlying type.

3. Any type that contains a go:notinheap type is itself go:notinheap. Structs and arrays are go:notinheap if their elements are. Maps and channels of go:notinheap types are disallowed. To keep things explicit, any type declaration where the type is implicitly go:notinheap must be explicitly marked go:notinheap as well.

4. Write barriers on pointers to go:notinheap types can be omitted.

The last point is the real benefit of go:notinheap. The runtime uses it for low-level internal structures to avoid memory barriers in the scheduler and the memory allocator where they are illegal or simply inefficient. This mechanism is reasonably safe and does not compromise the readability of the runtime.
```

嗯，了解一下就行了，在用户代码中基本不会用得到。

### 时间堆插入

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

func addtimer(t *timer) {
    tb := t.assignBucket()
    lock(&tb.lock)
    tb.addtimerLocked(t)
    unlock(&tb.lock)
}

func (t *timer) assignBucket() *timersBucket {
    id := uint8(getg().m.p.ptr().id) % timersLen
    t.tb = &timers[id].timersBucket
    return t.tb
}

// Add a timer to the heap and start or kick timerproc if the new timer is
// earlier than any of the others.
// Timers are locked.
func (tb *timersBucket) addtimerLocked(t *timer) {
    // when must never be negative; otherwise timerproc will overflow
    // during its delta calculation and never expire other runtime timers.
    if t.when < 0 {
        t.when = 1<<63 - 1
    }
    t.i = len(tb.t)
    tb.t = append(tb.t, t)
    siftupTimer(tb.t, t.i)
    if t.i == 0 {
        // siftup moved to top: new earliest deadline.
        if tb.sleeping {
            tb.sleeping = false
            notewakeup(&tb.waitnote)
        }
        if tb.rescheduling {
            tb.rescheduling = false
            goready(tb.gp, 0)
        }
    }
    if !tb.created {
        tb.created = true
        go timerproc(tb)
    }
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
