# panic 和 error 处理

## panic 的本质

```go
// The implementation of the predeclared function panic.
func gopanic(e interface{}) {
    gp := getg()
    if gp.m.curg != gp {
        print("panic: ")
        printany(e)
        print("\n")
        throw("panic on system stack")
    }

    // m.softfloat is set during software floating point.
    // It increments m.locks to avoid preemption.
    // We moved the memory loads out, so there shouldn't be
    // any reason for it to panic anymore.
    if gp.m.softfloat != 0 {
        gp.m.locks--
        gp.m.softfloat = 0
        throw("panic during softfloat")
    }
    if gp.m.mallocing != 0 {
        print("panic: ")
        printany(e)
        print("\n")
        throw("panic during malloc")
    }
    if gp.m.preemptoff != "" {
        print("panic: ")
        printany(e)
        print("\n")
        print("preempt off reason: ")
        print(gp.m.preemptoff)
        print("\n")
        throw("panic during preemptoff")
    }
    if gp.m.locks != 0 {
        print("panic: ")
        printany(e)
        print("\n")
        throw("panic holding locks")
    }

    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    atomic.Xadd(&runningPanicDefers, 1)

    for {
        d := gp._defer
        if d == nil {
            break
        }

        // If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
        // take defer off list. The earlier panic or Goexit will not continue running.
        if d.started {
            if d._panic != nil {
                d._panic.aborted = true
            }
            d._panic = nil
            d.fn = nil
            gp._defer = d.link
            freedefer(d)
            continue
        }

        // Mark defer as started, but keep on list, so that traceback
        // can find and update the defer's argument frame if stack growth
        // or a garbage collection happens before reflectcall starts executing d.fn.
        d.started = true

        // Record the panic that is running the defer.
        // If there is a new panic during the deferred call, that panic
        // will find d in the list and will mark d._panic (this panic) aborted.
        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        p.argp = unsafe.Pointer(getargp(0))
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
        p.argp = nil

        // reflectcall did not panic. Remove d.
        if gp._defer != d {
            throw("bad defer entry in panic")
        }
        d._panic = nil
        d.fn = nil
        gp._defer = d.link

        // trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
        //GC()

        pc := d.pc
        sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
        freedefer(d)
        if p.recovered {
            atomic.Xadd(&runningPanicDefers, -1)

            gp._panic = p.link
            // Aborted panics are marked but remain on the g.panic list.
            // Remove them from the list.
            for gp._panic != nil && gp._panic.aborted {
                gp._panic = gp._panic.link
            }
            if gp._panic == nil { // must be done with signal
                gp.sig = 0
            }
            // Pass information about recovering frame to recovery.
            gp.sigcode0 = uintptr(sp)
            gp.sigcode1 = pc
            mcall(recovery)
            throw("recovery failed") // mcall should not return
        }
    }

    // ran out of deferred calls - old-school panic now
    // Because it is unsafe to call arbitrary user code after freezing
    // the world, we call preprintpanics to invoke all necessary Error
    // and String methods to prepare the panic strings before startpanic.
    preprintpanics(gp._panic)
    startpanic()

    // startpanic set panicking, which will block main from exiting,
    // so now OK to decrement runningPanicDefers.
    atomic.Xadd(&runningPanicDefers, -1)

    printpanics(gp._panic)
    dopanic(0)       // should not return
    *(*int)(nil) = 0 // not reached
}

```

## error 处理
