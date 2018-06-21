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
