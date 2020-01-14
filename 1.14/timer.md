# timer

```go
type timer struct {
	pp puintptr // ** 新增字段

	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr

	nextwhen int64 // ** 新增字段，表示当 timer 的状态为 timerModifiedXX 时，下一次 when 应该设置为什么值，这里的 timerModifiedXX 包含 timerModifiedEarlier 和 timerModifiedLater 

	status uint32 // ** 新增字段，代表 timer 的状态，值为下面的枚举值
}
```

这里原文的注释写着：

```
// 本文件外的代码，在使用 timer 值时应该小心
//
//  pp，status 和 nextwhen 这几个新增加的字段只能在本文件内使用
//
// 创建 new timer 值时，只应该设置 when，period，f，arg 和 seq 这些字段
```

可以看出 Go 在字段权限控制上的一些弱势，只有较粗粒度的 pub 和 private，而没有办法做文件内的权限保护(因为当前目录(pkg)下的所有文件理论上可访问该 struct 的任意字段)。

创建 new timer 值之后，应该将该 timer 值传给 addtimer(实际是在 time.startTimer)中调用的。

一个活跃的 timer (即被传给 addtimer) 可能会被传给 deltimer (time.stopTimer)，这之后该 timer 便不是一个活跃的 timer。

在非活跃 timer 中，period，f，arg 和 seq 字段均会被修改，when 字段则不会发生修改。

删除一个非活跃的 timer 并让 GC 回收，这是没什么问题的。但把一个非活跃的 timer 传给 addtimer 就不 OK 了。

只有新创建的 timer 值才能允许传给 addtimer。

一个活跃的 timer 可能会传给 modtimer。其字段不会发生任何变动。该 timer 依然能够保持活跃。

一个非活跃的 timer 可能会被传给 resettimer 函数，被转换为一个活跃的 timer，并且会将其 when 字段进行更新。

新创建的 timer 也可以传给 resettimer。

timer 共有 addtimer，deltimer，modtimer，resettimer，cleantimers，adjusttimers 和 runtime 这些操作。

不允许同时调用 addtimer/deltimer/modtimer/resettimer。

但 adjusttimers 和 runtimer 可以同时与上述的任何操作同时调用。

活跃的 timer 在和 P 相关联的堆中，即 p 结构体的 timers 字段上。

非活跃的 timer 在被移除之前，也临时性地在 timers 堆中。

下面是 timer 进行各种操作时的可能状态迁移：

// TODO, draw status migration picture

```go
//
// addtimer:
//   timerNoStatus   -> timerWaiting
//   anything else   -> panic: invalid value
// deltimer:
//   timerWaiting    -> timerDeleted
//   timerModifiedXX -> timerDeleted
//   timerNoStatus   -> do nothing
//   timerDeleted    -> do nothing
//   timerRemoving   -> do nothing
//   timerRemoved    -> do nothing
//   timerRunning    -> wait until status changes
//   timerMoving     -> wait until status changes
//   timerModifying  -> panic: concurrent deltimer/modtimer calls
// modtimer:
//   timerWaiting    -> timerModifying -> timerModifiedXX
//   timerModifiedXX -> timerModifying -> timerModifiedYY
//   timerNoStatus   -> timerWaiting
//   timerRemoved    -> timerWaiting
//   timerRunning    -> wait until status changes
//   timerMoving     -> wait until status changes
//   timerRemoving   -> wait until status changes
//   timerDeleted    -> panic: concurrent modtimer/deltimer calls
//   timerModifying  -> panic: concurrent modtimer calls
// resettimer:
//   timerNoStatus   -> timerWaiting
//   timerRemoved    -> timerWaiting
//   timerDeleted    -> timerModifying -> timerModifiedXX
//   timerRemoving   -> wait until status changes
//   timerRunning    -> wait until status changes
//   timerWaiting    -> panic: resettimer called on active timer
//   timerMoving     -> panic: resettimer called on active timer
//   timerModifiedXX -> panic: resettimer called on active timer
//   timerModifying  -> panic: resettimer called on active timer
// cleantimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// adjusttimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// runtimer (looks in P's timer heap):
//   timerNoStatus   -> panic: uninitialized timer
//   timerWaiting    -> timerWaiting or
//   timerWaiting    -> timerRunning -> timerNoStatus or
//   timerWaiting    -> timerRunning -> timerWaiting
//   timerModifying  -> wait until status changes
//   timerModifiedXX -> timerMoving -> timerWaiting
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerRunning    -> panic: concurrent runtimer calls
//   timerRemoved    -> panic: inconsistent timer heap
//   timerRemoving   -> panic: inconsistent timer heap
//   timerMoving     -> panic: inconsistent timer heap
```

timer 的 status 字段可能有下面这些取值：

```
const (
	// Timer has no status set yet.
	timerNoStatus = iota

	// Waiting for timer to fire.
	// The timer is in some P's heap.
	timerWaiting

	// Running the timer function.
	// A timer will only have this status briefly.
	timerRunning

	// The timer is deleted and should be removed.
	// It should not be run, but it is still in some P's heap.
	timerDeleted

	// The timer is being removed.
	// The timer will only have this status briefly.
	timerRemoving

	// The timer has been stopped.
	// It is not in any P's heap.
	timerRemoved

	// The timer is being modified.
	// The timer will only have this status briefly.
	timerModifying

	// The timer has been modified to an earlier time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedEarlier

	// The timer has been modified to the same or a later time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedLater

	// The timer has been modified and is being moved.
	// The timer will only have this status briefly.
	timerMoving
)
```

## time.Sleep 的实现：

```go
//go:linkname timeSleep time.Sleep
func timeSleep(ns int64) {
	if ns <= 0 {
		return
	}

	gp := getg()
	t := gp.timer
	if t == nil {
		t = new(timer)
		gp.timer = t
	}
	t.f = goroutineReady
	t.arg = gp
	t.nextwhen = nanotime() + ns
	gopark(resetForSleep, unsafe.Pointer(t), waitReasonSleep, traceEvGoSleep, 1)
}

// resetForSleep 在 timeSleep 流程中，goroutine 被 parked 之后，便会调用
// 不能在 timeSleep 内直接调用 resettimer，因为如果 sleep 的参数是一个很小的时间值
// 且有很多 goroutine 的情况下，P 可能最终会在执行 gopark 之前就直接调用 timer 对象的 f，即 goroutineReady
func resetForSleep(gp *g, ut unsafe.Pointer) bool {
	t := (*timer)(ut)
	resettimer(t, t.nextwhen)
	return true
}
```

在 goroutine 被 parked 之后，会调用传入的 resetForSleep -> resettimer。

```go
func resettimer(t *timer, when int64) {
	for {
		switch s := atomic.Load(&t.status); s {
		case timerNoStatus, timerRemoved:
			if atomic.Cas(&t.status, s, timerWaiting) {
				t.when = when
				addInitializedTimer(t)
				return
            }
        }
    }
    ...
}
```

所以流程是 time.Sleep -> gopark -> resetForSleep -> resettimer -> addInitializedTimer -> (cleantimers && doaddtimer -> wakeNetPoller)。

相比老的 time.Sleep 实现，这里最大的变化是在 goroutine 被唤醒之前，没有锁了(？？？)。

```go
// 把 t 放进 timer heap
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
	if raceenabled {
		racerelease(unsafe.Pointer(t))
	}
	addtimer(t)
}

// 停止一个 timer
// 返回值表示 t 是否在被 run 之前就 stop 了
//go:linkname stopTimer time.stopTimer
func stopTimer(t *timer) bool {
	return deltimer(t)
}

// reset 一个非活跃的 timer，并把它塞进堆里
//go:linkname resetTimer time.resetTimer
func resetTimer(t *timer, when int64) {
	if raceenabled {
		racerelease(unsafe.Pointer(t))
	}
	resettimer(t, when)
}

// addtimer 给当前的 P 增加一个新的 timer
// 只有新创建的 timer 才应该使用该操作
// 这样可以避免修改某个 P 的堆上的 timer 的 when 字段，以避免导致 heap 从有序变为无序
func addtimer(t *timer) {
	// when must never be negative; otherwise runtimer will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = maxWhen
	}
	if t.status != timerNoStatus {
		badTimer()
	}
	t.status = timerWaiting

	addInitializedTimer(t)
}

// 给当前 P 增加一个已初始化的 timer
func addInitializedTimer(t *timer) {
	when := t.when

	pp := getg().m.p.ptr()
	lock(&pp.timersLock)
	ok := cleantimers(pp) && doaddtimer(pp, t)
	unlock(&pp.timersLock)
	if !ok {
		badTimer()
	}

	wakeNetPoller(when)
}

// doaddtimer adds t to the current P's heap.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func doaddtimer(pp *p, t *timer) bool {
	// Timers rely on the network poller, so make sure the poller
	// has started.
	if netpollInited == 0 {
		netpollGenericInit()
	}

	if t.pp != 0 {
		throw("doaddtimer: P already set in timer")
	}
	t.pp.set(pp)
	i := len(pp.timers)
	pp.timers = append(pp.timers, t)
	return siftupTimer(pp.timers, i)
}
```

```go
// deltimer deletes the timer t. It may be on some other P, so we can't
// actually remove it from the timers heap. We can only mark it as deleted.
// It will be removed in due course by the P whose heap it is on.
// Reports whether the timer was removed before it was run.
func deltimer(t *timer) bool {
	for {
		switch s := atomic.Load(&t.status); s {
		case timerWaiting, timerModifiedLater:
			if atomic.Cas(&t.status, s, timerDeleted) {
				// Timer was not yet run.
				return true
			}
		case timerModifiedEarlier:
			tpp := t.pp.ptr()
			if atomic.Cas(&t.status, s, timerModifying) {
				atomic.Xadd(&tpp.adjustTimers, -1)
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				// Timer was not yet run.
				return true
			}
		case timerDeleted, timerRemoving, timerRemoved:
			// Timer was already run.
			return false
		case timerRunning, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerNoStatus:
			// Removing timer that was never added or
			// has already been run. Also see issue 21874.
			return false
		case timerModifying:
			// Simultaneous calls to deltimer and modtimer.
			badTimer()
		default:
			badTimer()
		}
	}
}

// dodeltimer removes timer i from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer(pp *p, i int) bool {
	if t := pp.timers[i]; t.pp.ptr() != pp {
		throw("dodeltimer: wrong P")
	} else {
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if i != last {
		pp.timers[i] = pp.timers[last]
	}
	pp.timers[last] = nil
	pp.timers = pp.timers[:last]
	ok := true
	if i != last {
		// Moving to i may have moved the last timer to a new parent,
		// so sift up to preserve the heap guarantee.
		if !siftupTimer(pp.timers, i) {
			ok = false
		}
		if !siftdownTimer(pp.timers, i) {
			ok = false
		}
	}
	return ok
}

// dodeltimer0 removes timer 0 from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer0(pp *p) bool {
	if t := pp.timers[0]; t.pp.ptr() != pp {
		throw("dodeltimer0: wrong P")
	} else {
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if last > 0 {
		pp.timers[0] = pp.timers[last]
	}
	pp.timers[last] = nil
	pp.timers = pp.timers[:last]
	ok := true
	if last > 0 {
		ok = siftdownTimer(pp.timers, 0)
	}
	return ok
}
```

```go
// modtimer modifies an existing timer.
// This is called by the netpoll code.
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) {
	if when < 0 {
		when = maxWhen
	}

	status := uint32(timerNoStatus)
	wasRemoved := false
loop:
	for {
		switch status = atomic.Load(&t.status); status {
		case timerWaiting, timerModifiedEarlier, timerModifiedLater:
			if atomic.Cas(&t.status, status, timerModifying) {
				break loop
			}
		case timerNoStatus, timerRemoved:
			// Timer was already run and t is no longer in a heap.
			// Act like addtimer.
			if atomic.Cas(&t.status, status, timerWaiting) {
				wasRemoved = true
				break loop
			}
		case timerRunning, timerRemoving, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerDeleted:
			// Simultaneous calls to modtimer and deltimer.
			badTimer()
		case timerModifying:
			// Multiple simultaneous calls to modtimer.
			badTimer()
		default:
			badTimer()
		}
	}

	t.period = period
	t.f = f
	t.arg = arg
	t.seq = seq

	if wasRemoved {
		t.when = when
		addInitializedTimer(t)
	} else {
		// The timer is in some other P's heap, so we can't change
		// the when field. If we did, the other P's heap would
		// be out of order. So we put the new when value in the
		// nextwhen field, and let the other P set the when field
		// when it is prepared to resort the heap.
		t.nextwhen = when

		newStatus := uint32(timerModifiedLater)
		if when < t.when {
			newStatus = timerModifiedEarlier
		}

		// Update the adjustTimers field.  Subtract one if we
		// are removing a timerModifiedEarlier, add one if we
		// are adding a timerModifiedEarlier.
		tpp := t.pp.ptr()
		adjust := int32(0)
		if status == timerModifiedEarlier {
			adjust--
		}
		if newStatus == timerModifiedEarlier {
			adjust++
		}
		if adjust != 0 {
			atomic.Xadd(&tpp.adjustTimers, adjust)
		}

		// Set the new status of the timer.
		if !atomic.Cas(&t.status, timerModifying, newStatus) {
			badTimer()
		}

		// If the new status is earlier, wake up the poller.
		if newStatus == timerModifiedEarlier {
			wakeNetPoller(when)
		}
	}
}
```

```go
// resettimer resets an existing inactive timer to turn it into an active timer,
// with a new time for when the timer should fire.
// This should be called instead of addtimer if the timer value has been,
// or may have been, used previously.
func resettimer(t *timer, when int64) {
	if when < 0 {
		when = maxWhen
	}

	for {
		switch s := atomic.Load(&t.status); s {
		case timerNoStatus, timerRemoved:
			if atomic.Cas(&t.status, s, timerWaiting) {
				t.when = when
				addInitializedTimer(t)
				return
			}
		case timerDeleted:
			if atomic.Cas(&t.status, s, timerModifying) {
				t.nextwhen = when
				newStatus := uint32(timerModifiedLater)
				if when < t.when {
					newStatus = timerModifiedEarlier
					atomic.Xadd(&t.pp.ptr().adjustTimers, 1)
				}
				if !atomic.Cas(&t.status, timerModifying, newStatus) {
					badTimer()
				}
				if newStatus == timerModifiedEarlier {
					wakeNetPoller(when)
				}
				return
			}
		case timerRemoving:
			// Wait for the removal to complete.
			osyield()
		case timerRunning:
			// Even though the timer should not be active,
			// we can see timerRunning if the timer function
			// permits some other goroutine to call resettimer.
			// Wait until the run is complete.
			osyield()
		case timerWaiting, timerModifying, timerModifiedEarlier, timerModifiedLater, timerMoving:
			// Called resettimer on active timer.
			badTimer()
		default:
			badTimer()
		}
	}
}

```

```go
// cleantimers cleans up the head of the timer queue. This speeds up
// programs that create and delete timers; leaving them in the heap
// slows down addtimer. Reports whether no timer problems were found.
// The caller must have locked the timers for pp.
func cleantimers(pp *p) bool {
	for {
		if len(pp.timers) == 0 {
			return true
		}
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("cleantimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			if !dodeltimer0(pp) {
				return false
			}
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				return false
			}
		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			// Now we can change the when field.
			t.when = t.nextwhen
			// Move t to the right position.
			if !dodeltimer0(pp) {
				return false
			}
			if !doaddtimer(pp, t) {
				return false
			}
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				return false
			}
		default:
			// Head of timers does not need adjustment.
			return true
		}
	}
}
```

```go
// moveTimers moves a slice of timers to pp. The slice has been taken
// from a different P.
// This is currently called when the world is stopped, but the caller
// is expected to have locked the timers for pp.
func moveTimers(pp *p, timers []*timer) {
	for _, t := range timers {
	loop:
		for {
			switch s := atomic.Load(&t.status); s {
			case timerWaiting:
				t.pp = 0
				if !doaddtimer(pp, t) {
					badTimer()
				}
				break loop
			case timerModifiedEarlier, timerModifiedLater:
				if !atomic.Cas(&t.status, s, timerMoving) {
					continue
				}
				t.when = t.nextwhen
				t.pp = 0
				if !doaddtimer(pp, t) {
					badTimer()
				}
				if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
					badTimer()
				}
				break loop
			case timerDeleted:
				if !atomic.Cas(&t.status, s, timerRemoved) {
					continue
				}
				t.pp = 0
				// We no longer need this timer in the heap.
				break loop
			case timerModifying:
				// Loop until the modification is complete.
				osyield()
			case timerNoStatus, timerRemoved:
				// We should not see these status values in a timers heap.
				badTimer()
			case timerRunning, timerRemoving, timerMoving:
				// Some other P thinks it owns this timer,
				// which should not happen.
				badTimer()
			default:
				badTimer()
			}
		}
	}
}
```

```go
// adjusttimers looks through the timers in the current P's heap for
// any timers that have been modified to run earlier, and puts them in
// the correct place in the heap. While looking for those timers,
// it also moves timers that have been modified to run later,
// and removes deleted timers. The caller must have locked the timers for pp.
func adjusttimers(pp *p) {
	if len(pp.timers) == 0 {
		return
	}
	if atomic.Load(&pp.adjustTimers) == 0 {
		return
	}
	var moved []*timer
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		if t.pp.ptr() != pp {
			throw("adjusttimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			if atomic.Cas(&t.status, s, timerRemoving) {
				if !dodeltimer(pp, i) {
					badTimer()
				}
				if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
					badTimer()
				}
				// Look at this heap position again.
				i--
			}
		case timerModifiedEarlier, timerModifiedLater:
			if atomic.Cas(&t.status, s, timerMoving) {
				// Now we can change the when field.
				t.when = t.nextwhen
				// Take t off the heap, and hold onto it.
				// We don't add it back yet because the
				// heap manipulation could cause our
				// loop to skip some other timer.
				if !dodeltimer(pp, i) {
					badTimer()
				}
				moved = append(moved, t)
				if s == timerModifiedEarlier {
					if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
						addAdjustedTimers(pp, moved)
						return
					}
				}
			}
		case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
			badTimer()
		case timerWaiting:
			// OK, nothing to do.
		case timerModifying:
			// Check again after modification is complete.
			osyield()
			i--
		default:
			badTimer()
		}
	}

	if len(moved) > 0 {
		addAdjustedTimers(pp, moved)
	}
}

// addAdjustedTimers adds any timers we adjusted in adjusttimers
// back to the timer heap.
func addAdjustedTimers(pp *p, moved []*timer) {
	for _, t := range moved {
		if !doaddtimer(pp, t) {
			badTimer()
		}
		if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
			badTimer()
		}
	}
}

```

```go
// nobarrierWakeTime looks at P's timers and returns the time when we
// should wake up the netpoller. It returns 0 if there are no timers.
// This function is invoked when dropping a P, and must run without
// any write barriers. Therefore, if there are any timers that needs
// to be moved earlier, it conservatively returns the current time.
// The netpoller M will wake up and adjust timers before sleeping again.
//go:nowritebarrierrec
func nobarrierWakeTime(pp *p) int64 {
	lock(&pp.timersLock)
	ret := int64(0)
	if len(pp.timers) > 0 {
		if atomic.Load(&pp.adjustTimers) > 0 {
			ret = nanotime()
		} else {
			ret = pp.timers[0].when
		}
	}
	unlock(&pp.timersLock)
	return ret
}
```

```go
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//go:systemstack
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now {
				// Not ready to run.
				return t.when
			}

			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			runOneTimer(pp, t, now)
			return 0

		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			if !dodeltimer0(pp) {
				badTimer()
			}
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			if len(pp.timers) == 0 {
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			if !dodeltimer0(pp) {
				badTimer()
			}
			if !doaddtimer(pp, t) {
				badTimer()
			}
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield()

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}
```

```go
// runOneTimer runs a single timer.
// The caller must have locked the timers for pp.
// This will temporarily unlock the timers while running the timer function.
//go:systemstack
func runOneTimer(pp *p, t *timer, now int64) {
	if raceenabled {
		ppcur := getg().m.p.ptr()
		if ppcur.timerRaceCtx == 0 {
			ppcur.timerRaceCtx = racegostart(funcPC(runtimer) + sys.PCQuantum)
		}
		raceacquirectx(ppcur.timerRaceCtx, unsafe.Pointer(t))
	}

	f := t.f
	arg := t.arg
	seq := t.seq

	if t.period > 0 {
		// Leave in heap but adjust next time to fire.
		delta := t.when - now
		t.when += t.period * (1 + -delta/t.period)
		if !siftdownTimer(pp.timers, 0) {
			badTimer()
		}
		if !atomic.Cas(&t.status, timerRunning, timerWaiting) {
			badTimer()
		}
	} else {
		// Remove from heap.
		if !dodeltimer0(pp) {
			badTimer()
		}
		if !atomic.Cas(&t.status, timerRunning, timerNoStatus) {
			badTimer()
		}
	}

	if raceenabled {
		// Temporarily use the current P's racectx for g0.
		gp := getg()
		if gp.racectx != 0 {
			throw("runOneTimer: unexpected racectx")
		}
		gp.racectx = gp.m.p.ptr().timerRaceCtx
	}

	unlock(&pp.timersLock)

	f(arg, seq)

	lock(&pp.timersLock)

	if raceenabled {
		gp := getg()
		gp.racectx = 0
	}
}
```

```go
func timejump() *p {
	if faketime == 0 {
		return nil
	}

	// Nothing is running, so we can look at all the P's.
	// Determine a timer bucket with minimum when.
	var (
		minT    *timer
		minWhen int64
		minP    *p
	)
	for _, pp := range allp {
		if pp.status != _Pidle && pp.status != _Pdead {
			throw("non-idle P in timejump")
		}
		if len(pp.timers) == 0 {
			continue
		}
		c := pp.adjustTimers
		for _, t := range pp.timers {
			switch s := atomic.Load(&t.status); s {
			case timerWaiting:
				if minT == nil || t.when < minWhen {
					minT = t
					minWhen = t.when
					minP = pp
				}
			case timerModifiedEarlier, timerModifiedLater:
				if minT == nil || t.nextwhen < minWhen {
					minT = t
					minWhen = t.nextwhen
					minP = pp
				}
				if s == timerModifiedEarlier {
					c--
				}
			case timerRunning, timerModifying, timerMoving:
				badTimer()
			}
			// The timers are sorted, so we only have to check
			// the first timer for each P, unless there are
			// some timerModifiedEarlier timers. The number
			// of timerModifiedEarlier timers is in the adjustTimers
			// field, used to initialize c, above.
			if c == 0 {
				break
			}
		}
	}

	if minT == nil || minWhen <= faketime {
		return nil
	}

	faketime = minWhen
	return minP
}

// timeSleepUntil returns the time when the next timer should fire.
// This is only called by sysmon.
func timeSleepUntil() int64 {
	next := int64(maxWhen)

	// Prevent allp slice changes. This is like retake.
	lock(&allpLock)
	for _, pp := range allp {
		if pp == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}

		lock(&pp.timersLock)
		c := atomic.Load(&pp.adjustTimers)
		for _, t := range pp.timers {
			switch s := atomic.Load(&t.status); s {
			case timerWaiting:
				if t.when < next {
					next = t.when
				}
			case timerModifiedEarlier, timerModifiedLater:
				if t.nextwhen < next {
					next = t.nextwhen
				}
				if s == timerModifiedEarlier {
					c--
				}
			}
			// The timers are sorted, so we only have to check
			// the first timer for each P, unless there are
			// some timerModifiedEarlier timers. The number
			// of timerModifiedEarlier timers is in the adjustTimers
			// field, used to initialize c, above.
			//
			// We don't worry about cases like timerModifying.
			// New timers can show up at any time,
			// so this function is necessarily imprecise.
			// Do a signed check here since we aren't
			// synchronizing the read of pp.adjustTimers
			// with the check of a timer status.
			if int32(c) <= 0 {
				break
			}
		}
		unlock(&pp.timersLock)
	}
	unlock(&allpLock)

	return next
}
```

siftUpTimer 和 siftDownTimer 这两个函数的实现和以前没什么不同，只是去除了冗余的 index 赋值操作。
