# Garbage Collection

## 初始化

```go
// Initialized from $GOGC.  GOGC=off means no GC.
var gcpercent int32

func gcinit() {
    if unsafe.Sizeof(workbuf{}) != _WorkbufSize {
        throw("size of Workbuf is suboptimal")
    }

    // No sweep on the first cycle.
    mheap_.sweepdone = 1

    // Set a reasonable initial GC trigger.
    memstats.triggerRatio = 7 / 8.0

    // Fake a heap_marked value so it looks like a trigger at
    // heapminimum is the appropriate growth from heap_marked.
    // This will go into computing the initial GC goal.
    memstats.heap_marked = uint64(float64(heapminimum) / (1 + memstats.triggerRatio))

    // Set gcpercent from the environment. This will also compute
    // and set the GC trigger and goal.
    _ = setGCPercent(readgogc())

    work.startSema = 1
    work.markDoneSema = 1
}

func readgogc() int32 {
    p := gogetenv("GOGC")
    if p == "off" {
        return -1
    }
    if n, ok := atoi32(p); ok {
        return n
    }
    return 100
}

// gcenable is called after the bulk of the runtime initialization,
// just before we're about to start letting user code run.
// It kicks off the background sweeper goroutine and enables GC.
func gcenable() {
    c := make(chan int, 1)
    go bgsweep(c)
    <-c
    memstats.enablegc = true // now that runtime is initialized, GC is okay
}

//go:linkname setGCPercent runtime/debug.setGCPercent
func setGCPercent(in int32) (out int32) {
    lock(&mheap_.lock)
    out = gcpercent
    if in < 0 {
        in = -1
    }
    gcpercent = in
    heapminimum = defaultHeapMinimum * uint64(gcpercent) / 100
    // Update pacing in response to gcpercent change.
    gcSetTriggerRatio(memstats.triggerRatio)
    unlock(&mheap_.lock)

    // If we just disabled GC, wait for any concurrent GC to
    // finish so we always return with no GC running.
    if in < 0 {
        // Disable phase transitions.
        lock(&work.sweepWaiters.lock)
        if gcphase == _GCmark {
            // GC is active. Wait until we reach sweeping.
            gp := getg()
            gp.schedlink = work.sweepWaiters.head
            work.sweepWaiters.head.set(gp)
            goparkunlock(&work.sweepWaiters.lock, "wait for GC cycle", traceEvGoBlock, 1)
        } else {
            // GC isn't active.
            unlock(&work.sweepWaiters.lock)
        }
    }

    return out
}
```

## runtime.GC

```go
// GC runs a garbage collection and blocks the caller until the
// garbage collection is complete. It may also block the entire
// program.
func GC() {
    // We consider a cycle to be: sweep termination, mark, mark
    // termination, and sweep. This function shouldn't return
    // until a full cycle has been completed, from beginning to
    // end. Hence, we always want to finish up the current cycle
    // and start a new one. That means:
    //
    // 1. In sweep termination, mark, or mark termination of cycle
    // N, wait until mark termination N completes and transitions
    // to sweep N.
    //
    // 2. In sweep N, help with sweep N.
    //
    // At this point we can begin a full cycle N+1.
    //
    // 3. Trigger cycle N+1 by starting sweep termination N+1.
    //
    // 4. Wait for mark termination N+1 to complete.
    //
    // 5. Help with sweep N+1 until it's done.
    //
    // This all has to be written to deal with the fact that the
    // GC may move ahead on its own. For example, when we block
    // until mark termination N, we may wake up in cycle N+2.

    gp := getg()

    // Prevent the GC phase or cycle count from changing.
    lock(&work.sweepWaiters.lock)
    n := atomic.Load(&work.cycles)
    if gcphase == _GCmark {
        // Wait until sweep termination, mark, and mark
        // termination of cycle N complete.
        gp.schedlink = work.sweepWaiters.head
        work.sweepWaiters.head.set(gp)
        goparkunlock(&work.sweepWaiters.lock, "wait for GC cycle", traceEvGoBlock, 1)
    } else {
        // We're in sweep N already.
        unlock(&work.sweepWaiters.lock)
    }

    // We're now in sweep N or later. Trigger GC cycle N+1, which
    // will first finish sweep N if necessary and then enter sweep
    // termination N+1.
    gcStart(gcBackgroundMode, gcTrigger{kind: gcTriggerCycle, n: n + 1})

    // Wait for mark termination N+1 to complete.
    lock(&work.sweepWaiters.lock)
    if gcphase == _GCmark && atomic.Load(&work.cycles) == n+1 {
        gp.schedlink = work.sweepWaiters.head
        work.sweepWaiters.head.set(gp)
        goparkunlock(&work.sweepWaiters.lock, "wait for GC cycle", traceEvGoBlock, 1)
    } else {
        unlock(&work.sweepWaiters.lock)
    }

    // Finish sweep N+1 before returning. We do this both to
    // complete the cycle and because runtime.GC() is often used
    // as part of tests and benchmarks to get the system into a
    // relatively stable and isolated state.
    for atomic.Load(&work.cycles) == n+1 && gosweepone() != ^uintptr(0) {
        sweep.nbgsweep++
        Gosched()
    }

    // Callers may assume that the heap profile reflects the
    // just-completed cycle when this returns (historically this
    // happened because this was a STW GC), but right now the
    // profile still reflects mark termination N, not N+1.
    //
    // As soon as all of the sweep frees from cycle N+1 are done,
    // we can go ahead and publish the heap profile.
    //
    // First, wait for sweeping to finish. (We know there are no
    // more spans on the sweep queue, but we may be concurrently
    // sweeping spans, so we have to wait.)
    for atomic.Load(&work.cycles) == n+1 && atomic.Load(&mheap_.sweepers) != 0 {
        Gosched()
    }

    // Now we're really done with sweeping, so we can publish the
    // stable heap profile. Only do this if we haven't already hit
    // another mark termination.
    mp := acquirem()
    cycle := atomic.Load(&work.cycles)
    if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
        mProf_PostSweep()
    }
    releasem(mp)
}
```