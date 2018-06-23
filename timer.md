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

### 四叉小顶堆性质

四叉堆高度上比二叉堆要矮一些。直接孩子节点一定比父节点大，同一层的相邻四个节点是按从小到大排列的，但如果不相邻，那么就不遵循这样的关系，所以，下面这种情况是符合四叉小顶堆的性质的:

```
                                                             ┌─────┐                                                         
                                                             │     │                                                         
                                                             │  0  │                                                         
                                                             └─────┘                                                         
                                                                │                                                            
                                                                │                                                            
                                                                │                                                            
                                                                ▼                                                            
                                                    ┌─────┬─────┬─────┬─────┐                                                
                                                    │     │     │     │     │                                                
                                                    │  1  │  2  │  7  │  11 │                                                
                                                    └─────┴─────┴─────┴─────┘                                                
                                                       │     │     │     │                                                   
                                                       │     │     │     │                                                   
                    ┌──────────┐                       │     │     │     │                                                   
   ┌────────────────┤  4*i+1   ├───────────────────────┘     │     │     └─────────────────────────────┐                     
   │                └──────────┘         ┌───────────────────┘     └───┐                               │                     
   │                                     │                             │                               │                     
   │                                     │                             │                               │                     
   ▼                                     │                             │                               ▼                     
┌─────┬─────┬─────┬─────┐                │                             │                            ┌─────┬─────┬─────┬─────┐
│     │     │     │     │                ▼                             ▼                            │     │     │     │     │
│  13 │  15 │  17 │  20 │             ┌─────┬─────┬─────┬─────┐     ┌─────┬─────┬─────┬─────┐       │ 12  │ 13  │ 15  │  16 │
└─────┴─────┴─────┴─────┘             │     │     │     │     │     │     │     │     │     │       └─────┴─────┴─────┴─────┘
                                      │ 12  │ 14  │ 15  │  16 │     │ 8   │ 9   │ 11  │  12 │                                
                                      └─────┴─────┴─────┴─────┘     └─────┴─────┴─────┴─────┘                                
```

当然，二叉小顶堆同一层的大小关系也是不确定的。只是四叉堆在同一层还有这种特殊的相邻的情况。

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
        // 同一个 P 上的所有 timers 如果
        // 都在 timerproc 中被弹出了
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

插入 timer 到堆中的时候的逻辑是先追加到数组末尾(append)，然后向上 adjust(siftup) heap 以重新恢复四叉小顶堆性质。

### 时间堆删除

```go
// 从堆中删除 timer t
// 如果 timerproc 提前被唤醒也没所谓
func deltimer(t *timer) bool {
    if t.tb == nil {
        // t.tb can be nil if the user created a timer
        // directly, without invoking startTimer e.g
        //    time.Ticker{C: c}
        // In this case, return early without any deletion.
        // See Issue 21874.
        return false
    }

    tb := t.tb

    lock(&tb.lock)
    // t may not be registered anymore and may have
    // a bogus i (typically 0, if generated by Go).
    // Verify it before proceeding.
    i := t.i
    last := len(tb.t) - 1
    if i < 0 || i > last || tb.t[i] != t {
        unlock(&tb.lock)
        return false
    }
    // 把 timer[i] 替换为 timer[last]
    if i != last {
        tb.t[i] = tb.t[last]
        tb.t[i].i = i
    }
    // 删除 timer[last]，并缩小 slice
    tb.t[last] = nil
    tb.t = tb.t[:last]

    // 判断是不是删的最后一个
    // 如果不是的话，需要重新调整堆
    if i != last {
        // 最后一个节点当前来的分叉可能并不是它那个分叉
        // 所以向上走或者向下走都是有可能的
        // 即使是二叉堆，也是有这种可能的
        siftupTimer(tb.t, i)
        siftdownTimer(tb.t, i)
    }
    unlock(&tb.lock)
    return true
}
```

### timer 触发

```go
// timerproc 负责处理时间驱动的事件
// 在堆中的下一个事件需要触发之前，会一直保持 sleep 状态
// 如果 addtimer 插入了一个更早的事件，会提前唤醒 timerproc
func timerproc(tb *timersBucket) {
    tb.gp = getg()
    for {
        // timerBucket 的局部大锁
        lock(&tb.lock)
        // 被唤醒，所以修改 sleeping 状态为 false
        tb.sleeping = false
        // 计时
        now := nanotime()
        delta := int64(-1)
        // 在处理完到期的 timer 之前，一直循环
        for {
            // 如果 timer 已经都弹出了
            // 那么不用循环了，跳出后接着睡觉
            if len(tb.t) == 0 {
                delta = -1
                break
            }
            // 取小顶堆顶部元素
            // 即最近会被触发的 timer
            t := tb.t[0]
            delta = t.when - now // 还差多长时间才需要触发最近的 timer
            if delta > 0 {
                // 大于 0 说明这个 timer 还没到需要触发的时间
                // 跳出循环去睡觉
                break
            }
            if t.period > 0 {
                // 这个 timer 还会留在堆里
                // 不过要调整它的下次触发时间
                t.when += t.period * (1 + -delta/t.period)
                siftdownTimer(tb.t, 0)
            } else {
                // 从堆中移除这个 timer
                // 用最后一个 timer 覆盖第 0 个 timer
                // 然后向下调整堆
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
                t.i = -1 // 标记 timer 在堆中的位置已经没有了
            }
            // timer 触发时需要调用的函数
            f := t.f
            arg := t.arg
            seq := t.seq
            unlock(&tb.lock)
            // 调用需触发的函数
            f(arg, seq)
            // 把锁加回来，如果下次 break 了内层的 for 循环
            // 能保证 timeBucket 是被锁住的
            // 然后在下面的 goparkunlock 中被解锁
            lock(&tb.lock)
        }
        if delta < 0 || faketime > 0 {
            // 说明时间堆里已经没有 timer 了
            // 让 goroutine 挂起，去睡觉
            tb.rescheduling = true
            goparkunlock(&tb.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
            continue
        }
        // 说明堆里至少还有一个以上的 timer
        // 睡到最近的 timer 时间
        tb.sleeping = true
        tb.sleepUntil = now + delta
        noteclear(&tb.waitnote)
        unlock(&tb.lock)
        // 内部是 futex sleep
        // 时间睡到了会自动醒
        // 或者 addtimer 的时候，发现新的 timer 更早，会提前唤醒
        notetsleepg(&tb.waitnote, delta)
    }
}
```

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

```

TODO timejump